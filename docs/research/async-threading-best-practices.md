# Async + Threading Runtime Design: Research for Riven v2

Audience: principal engineer designing v2 of Riven's async runtime. Assumes async + threads conceptually. Focus: the specific choices production runtimes have converged on, and where Riven's current shape diverges.

---

## 1. The async runtime design space, 2024-2026

The interesting axis is not "async vs threads" but **how many reactors, where do tasks live, and how do wakes cross threads**. The runtimes worth studying have settled into three camps.

### 1.1 Tokio (Rust) — M:N work-stealing, shared reactor topology

- **Scheduler model**: M:N work-stealing. `N = num_cpus` worker threads by default. Each worker owns a fixed-size LIFO local run queue (capacity 256 tasks) that overflows to a global FIFO queue. The local queue is LIFO for cache-locality on message-passing patterns; the **stealable view** of it is FIFO so older tasks drain fairly to idle workers ([tokio multi_thread worker](https://docs.rs/tokio/latest/src/tokio/runtime/scheduler/multi_thread/worker.rs.html)).
- **LIFO slot**: each worker has a single-task "next" slot that bypasses the queue entirely; ping-pong message patterns stay on one core. This slot is **not stealable**, which can starve long-poll tasks.
- **Steal granularity**: half of the victim's queue, not one task — reduces "thrash and re-steal" ([Making the Tokio scheduler 10x faster](https://tokio.rs/blog/2019-10-scheduler)).
- **Searcher throttle**: at most `num_cpus / 2` workers may be in "searching" state simultaneously, capping the cost of contention on victim queues.
- **Parking**: a worker that finds nothing tries a bounded number of steal attempts (with backoff), then registers itself as parked and calls `epoll_wait` / `kevent` if it's the designated "I/O worker" — otherwise it `cond_wait`s. Wake from another thread atomically increments a counter and writes one byte to an **eventfd** (Linux) or pipes/kqueue-user-event (other platforms) to interrupt `epoll_wait` ([mio waker](https://github.com/tokio-rs/mio/blob/master/src/waker.rs)).
- **Reactor topology**: a **single shared mio `Selector`** (one epoll/kqueue fd) for the whole runtime, owned by the I/O driver. Workers do not each own a reactor; they share one. Per-fd registration state is held in a `Sync` slab and freed when the `Registration` is dropped ([Tokio I/O driver](https://deepwiki.com/tokio-rs/tokio/3.3-notification-and-signaling)).
- **Send constraints**: `tokio::spawn` requires `Future: Send + 'static` so tasks can move across workers. `spawn_local` exists but only on the `LocalSet` / current-thread runtime.
- **Cross-thread wake**: `AtomicWaker` (single producer-cell, multi-consumer-race protected by a 2-bit state machine) sits inside every task; calling `wake` does a CAS on the task's scheduler-ref then pushes to a worker's injection queue and writes the eventfd if the worker is parked ([AtomicWaker source](https://github.com/tokio-rs/tokio/blob/master/tokio/src/sync/task/atomic_waker.rs)).

### 1.2 Go (goroutines + netpoll) — M:N with kernel-level parking

- **GMP model**: M (OS thread) acquires a P (logical processor; `GOMAXPROCS` of them) to run a G (goroutine). Each P has a local run queue (256 G's), with overflow to a global queue. Work-stealing across P's; stolen batch is half the victim's queue.
- **Reactor**: a single shared netpoller (epoll on Linux, kqueue on macOS, IOCP on Windows). When a P's queues are empty and no other M is currently polling, that M calls `netpoll(block=true)` which means `epoll_wait` ([Go netpoll](https://go.dev/src/runtime/netpoll.go)).
- **Parking**: goroutine blocking on I/O transitions to `_Gwaiting`; the M releases its P and parks; readiness moves the G to `_Grunnable` and into a P's queue. Scheduler points are inserted by the compiler at function calls (cooperative, not preemptive until 1.14's async preempt via signals).
- **Stack**: goroutines start at **2 KB** with a copying-stack growth strategy. Cheap to spawn millions.

### 1.3 libuv (Node.js) — single-threaded event loop, thread pool for blocking work

- One event loop per process by default. All JS runs on one thread. Blocking work (fs, DNS, user-supplied) goes to a 4-thread libuv worker pool. No work-stealing — phase-ordered queues (timers, pending, idle, poll, check, close).
- Node clustering = fork N processes, each with its own loop, bound via SO_REUSEPORT or a master-dispatch socket.

### 1.4 Erlang BEAM — one scheduler per core, reduction-counted preemption

- N schedulers, default = number of online CPUs. Each scheduler has its own run queue and steals from peers. Migration logic balances queues every ~16ms ([Deep dive into the Erlang scheduler](https://blog.appsignal.com/2024/04/23/deep-diving-into-the-erlang-scheduler.html)).
- **Reductions**: every BIF and function call decrements a counter (default 4000); on exhaustion the process is preempted. This is what makes BEAM truly preemptive without OS signal tricks.
- **I/O polling**: since OTP 21, polling moved off the scheduler threads onto dedicated **I/O poll threads**. Default count is `ceil(schedulers * 0.25)`; configurable via `+IOPt`. This was a deliberate move *away* from "scheduler-thread also polls" — they found scheduler-side polling was adding latency variance.

### 1.5 Java Project Loom — virtual threads on a ForkJoinPool

- Virtual threads multiplex on `N = parallelism` (default `num_cpus`) carrier threads (OS threads) via a `ForkJoinPool` with work-stealing.
- Blocking call inside a virtual thread unmounts the continuation, parks it on a waker, frees the carrier. Re-mount on resume.
- Cross-thread wake via `LockSupport.unpark`, which on Linux is futex.

### 1.6 Thread-per-core: glommio, monoio, seastar

- **One executor + one reactor per thread, no sharing**. Tasks are `!Send`. No work-stealing.
- Glommio uses io_uring; monoio uses io_uring with a Tokio-compatible API; seastar (C++) uses kqueue/epoll/io_uring.
- Cross-thread communication via explicit shard-routed channels (one MPSC per (src,dst) pair) — no implicit task migration.
- **Benchmark claims**: monoio reports 2x tokio throughput at 4 cores, 3x at 16 cores on networking benchmarks; its gateway prototype beat NGINX by ~20% ([monoio comparison](https://zread.ai/bytedance/monoio/30-comparing-with-tokio-and-glommio)).
- Trade-off: any imbalance (one shard with a slow handler, an uneven hash) is unrecoverable. You buy lower variance and higher peak throughput by giving up automatic load balancing.

### Axis summary

| Runtime | Sched | Tasks→Threads | Send req | Reactor | Cross-thread wake |
|---|---|---|---|---|---|
| tokio (multi-thread) | work-stealing | M:N | yes | 1 shared | eventfd/kqueue-user |
| tokio (current-thread) | none | N:1 | no | 1 local | none needed |
| Go | work-stealing | M:N | yes (implicit) | 1 shared | futex via runtime |
| libuv | phased loop | N:1 per process | no | 1 per loop | uv_async (eventfd) |
| BEAM | per-scheduler + migration | M:N | yes (copy-only msgs) | dedicated I/O threads | futex / SIGURG |
| Loom | work-stealing FJP | M:N | yes | nio Selector(s) | LockSupport (futex) |
| glommio/monoio | thread-per-core | 1:1 per thread | **no** | 1 per thread | explicit channel + eventfd |

### Backpressure

Universal: bounded queues somewhere. tokio's `mpsc` is bounded; Go's channels are bounded; BEAM mailboxes are unbounded by default (the famous source of outages — use `gen_server` with `{:noreply, state, :hibernate}` and selective receive, or a `:queue` with explicit drop). Riven currently has **no backpressure anywhere** — the accept loop is the only natural throttle, and it's a tight loop.

---

## 2. Where Riven sits today

Mapped onto the axes:

- **Scheduling**: N independent **current-thread** executors (one per OS worker), no shared state. Effectively a static **event-loop-per-thread** with kernel load-balancing via SO_REUSEPORT. Closest cousin: libuv-clustered or a minimal seastar.
- **Task↔thread**: 1:1. A future runs to completion on the thread that started it. Not Send. There is no "task" abstraction — `block_on` drives a single future to completion synchronously.
- **Reactor**: **one per thread**, in TLS. Lazy-init on first `block_on`. **Never released** until thread exit (a deliberate fix for an EMFILE bug under churn). One leaked kqueue/epoll fd per ever-active thread. At 4 workers this is 4 fds — irrelevant. It would matter only if Riven ever spawns/destroys workers dynamically.
- **Wake**: no-op. The reactor blocks in `kevent`/`epoll_wait` between polls; the AST-rewriter loop polls in a tight loop between yields. **No cross-thread wake exists** because no future is ever woken from another thread.
- **I/O loop shape**: per-request, 3 `block_on` calls. Each does at minimum a `kevent_register → kevent_wait → kevent_deregister` triple. That's 3 syscalls of overhead per I/O step, 9 syscalls per request floor, on top of `accept/read/write/close`.

What it cannot do today, no matter how much you tune:
1. Move a slow request off a hot worker. SO_REUSEPORT load balancing is hash-based; one stuck handler stalls 1/N of new connections until the kernel times out the listen queue.
2. Wake a future from background work. There is no waker; the architecture assumes the only thing that wakes you is the reactor.
3. Avoid the register/deregister churn per I/O. tokio's mio keeps the fd registered for the lifetime of the `AsyncFd` and just re-arms readiness; Riven re-registers per future.

**This last point is almost certainly where the 40k vs 42k gap is hiding.** See section 4.

---

## 3. Decisions Riven should make for v2

### 3.1 Stay thread-per-core. Do not adopt work-stealing.

**Decision**: keep N independent event loops, one per OS thread. No cross-thread task migration. Tasks remain `!Send`. Match the **monoio / seastar / nginx / envoy** shape, not tokio's default.

**Why**: Riven's target workload is HTTP request/response, the language is young, the runtime is C-backed, and the FFI surface is small. Work-stealing requires (a) `Send` futures (b) `Sync` waker storage (c) atomic queues and (d) careful Drop semantics across threads. Drop-elaboration in Riven is "still maturing" per the project notes — that single fact rules out work-stealing for v2. Every runtime that adopted work-stealing paid for it with a multi-year correctness debt (tokio's `JoinHandle` cancellation bugs, Go's pre-1.14 preemption hangs). You don't want that bill yet.

The thread-per-core renaissance (glommio 2020, monoio 2021, seastar going back to 2014) is partly philosophical, but the load-distribution math is real: on uniform HTTP workloads, SO_REUSEPORT + sharded reactors gives within 5-10% of work-stealing throughput at a fraction of the implementation complexity ([Datadog glommio post](https://www.datadoghq.com/blog/engineering/introducing-glommio/)).

**Second-best**: a single-thread tokio-style runtime with `LocalSet` semantics + a `spawn_blocking` thread pool. Cost: you have to design and implement a task abstraction (boxed type-erased futures, a slab for storage, a runqueue). That's 3-6 months of work for a marginal win on this workload.

### 3.2 Single accept, dispatch via SPSC ring — only if you outgrow SO_REUSEPORT.

**Decision (v2)**: keep SO_REUSEPORT. Don't add a dispatcher thread yet.

**Why**: SO_REUSEPORT was Linux's answer to nginx's pre-existing single-accept-with-EPOLLEXCLUSIVE problem. Envoy, nginx 1.9+, and HAProxy all default to it ([Envoy listener balancing](https://medium.com/@hnasr/envoys-listener-connection-balancing-e015a9b09a85)). Kernel hashing is by 4-tuple; for HTTP/1.1 with many short connections it's fairly even. Known failure mode: Linux's BPF-less REUSEPORT can leave one worker idle if connections are very long-lived and few — Cloudflare documented this ([sad state of Linux socket balancing](https://blog.cloudflare.com/the-sad-state-of-linux-socket-balancing/)). For HTTP/1.0-close at 40k RPS, you're not in that regime.

The "single accept, dispatch via SPSC to N workers" model is what envoy uses with its "exact balance" listener option, and what some Go programs do with hand-rolled accept dispatching. It guarantees perfect balance at the cost of one cross-thread handoff per connection (~50-200ns and a cache line bounce). For Riven, it's a v3 lever to pull *if* you see imbalance under measured load.

### 3.3 Keep the reactor leaked. Document it.

**Decision**: the current "one reactor per thread, never closed" model is correct. Do not try to be clever.

**Why**: tokio handles this by closing the mio `Selector` (and thus the epoll fd) when the entire `Runtime` is dropped, which in practice means at process shutdown. mio's `Selector::Drop` calls `close()` on the epoll fd ([mio epoll selector](https://github.com/tokio-rs/mio/blob/master/src/sys/unix/selector/epoll.rs)). The reason your per-`block_on` close was buggy is that you were doing it at the wrong granularity — kqueue fd creation is not free, and tearing it down between blocks while connections were still alive is what made you trip EMFILE.

The correct model: **reactor lifetime = thread lifetime**. Document this invariant. If you ever add dynamic worker pools, register a `pthread_cleanup_push` that closes the reactor fd on thread exit. Otherwise, the OS reclaims it at process exit.

### 3.4 Add a real waker. Use eventfd on Linux, EVFILT_USER on macOS.

**Decision**: every reactor gets a wake fd registered at level-triggered for read. The waker for a task holds (reactor_ptr, task_id). `wake()` writes 8 bytes to the wake fd. The reactor's `epoll_wait` returns, and the runtime polls the woken task.

On macOS, use `EV_SET(&kev, ident, EVFILT_USER, EV_ADD|EV_CLEAR, 0, 0, NULL)` and trigger with `NOTE_TRIGGER`. This is exactly what mio does ([mio waker source](https://github.com/tokio-rs/mio/blob/master/src/waker.rs)). USR1 signals are wrong for this — signal delivery is per-process, not per-thread, and you'd need `pthread_kill` + a signal handler that does an `eventfd_write`-equivalent. Just use eventfd.

**Why this matters even in thread-per-core**: timers, channel sends from `spawn_blocking` work, and any future composed from non-I/O sources need a way to schedule themselves. Without a waker, you're forced to make every wake go through an I/O readiness event, which is what your current "tight poll loop between yields" is — and it costs you CPU.

### 3.5 Keep the fd registered for the lifetime of the AsyncTcpStream. Stop re-registering per future.

**Decision**: an `AsyncTcpStream` registers its fd with the reactor on construction (or first I/O) and deregisters on `Drop`. Read/write futures only update the readiness interest mask and store their waker. They do **not** call `kevent(EV_ADD)` per poll.

**Why**: this is the dominant per-request cost on your current shape. 3× `block_on` × (register + deregister) = 6 syscalls per request that tokio doesn't pay. At your 40k RPS measurement, that's 240k syscalls/sec of register/deregister churn on top of actual I/O. Removing it is probably worth 5-15k RPS by itself.

tokio's `AsyncFd` model: one `epoll_ctl(ADD)` at construction, one `epoll_ctl(DEL)` at drop. Readiness is edge-triggered (`EPOLLET`) so the kernel doesn't redundantly report. Futures poll until EAGAIN, then store the waker and yield.

### 3.6 Stack size: 256 KB is too small.

**Decision**: ship 512 KB or 1 MB for v2, configurable via the `Thread.spawn` builder.

**Why**: pthread default on Linux is 8 MB; on macOS, 512 KB (non-main thread) or 8 MB (main). 256 KB is below macOS default and **at the edge** of "fine for HTTP handlers, dangerous for anything that calls into user code with non-trivial stack frames." Riven handlers will call user code through FFI; user code may recurse, allocate large structs on the stack, or call into LLVM-generated code with high stack usage. 1 MB costs you 768 KB of address space per worker — at 4 workers, 3 MB. Irrelevant. The cost of getting a stack overflow at 256 KB in production is a SIGSEGV with no useful trace.

nginx workers run on the OS default (8 MB virtual, paged in lazily). Go starts goroutines at 2 KB but **only because Go can move stacks**, which Riven cannot. envoy uses pthread defaults. **You are not goroutines. Be honest about your stack story and ship a real number.**

### 3.7 HTTP/1.1 keepalive: this is the biggest single RPS win available.

**Decision**: implement HTTP/1.1 keepalive (`Connection: keep-alive`, persistent-by-default per RFC 7230) before any other Rondo perf work.

**Why**: close-per-connection at 40k RPS means 40k three-way handshakes/sec, 40k FIN/ACK sequences, 40k accept syscalls, 40k TIME_WAIT entries accumulating in the kernel. Keepalive amortizes accept+close across the connection's request count. Production HTTP servers see 5-10x RPS improvement going from close-per-request to keepalive on local-network benchmarks ([F5 on HTTP keepalive](https://www.f5.com/company/blog/nginx/http-keepalives-and-web-performance)). The TechEmpower plaintext benchmark (the closest analog to your "single-route GET") is **explicitly defined with HTTP pipelining over keepalive** — that's how axum/actix hit 7M+ RPS on 28-core systems. Without keepalive, you are measuring the kernel's TCP stack, not your framework.

Plumbing: read until end-of-headers, parse `Content-Length`/`Transfer-Encoding`, drain body, write response, **don't close**, loop. Add an idle timeout (15-75s — nginx default is 75, envoy 60). Track per-connection state (you'll need a tiny state machine: HEADERS → BODY → DRAIN → IDLE).

### 3.8 fd lifecycle: RAII via Drop on AsyncTcpStream. No explicit close.

**Decision**: `AsyncTcpStream::drop` calls `close(fd)`. The deregistration from the reactor happens **first**, then close. Mirror tokio's `AsyncFd::Drop` order.

**Why**: closing a registered fd is the most fertile bug-source in epoll/kqueue code. On Linux, `close()` on a `dup`'d fd does *not* remove the registration (the kernel epoll keeps the underlying struct file alive). On macOS, kqueue removes registrations on close, but only for the last reference. The safe order is always **deregister → close**. RAII via Drop makes this automatic and impossible to skip, *provided* drop elaboration runs reliably. If drop elaboration is shaky in Riven today, document a `close()` method as the explicit-cleanup escape hatch and have the framework call it before drop — belt and suspenders until the compiler matures.

---

## 4. Bench reality check

Your 40k RPS on a 4-worker M-series macOS rondo-async with HTTP/1.0-close is a measurement of **the macOS kernel's TCP accept/close path under tight contention, not your framework**.

For context (TechEmpower Round 23, plaintext, properly tuned: HTTP/1.1 pipelining, keepalive on, on 28-core Xeon, Linux):

- actix-web (Rust, tokio): ~7,000,000 RPS
- axum (Rust, tokio): ~6,500,000 RPS
- hyper (Rust, tokio, raw): ~7,200,000 RPS
- Go net/http: ~1,000,000 RPS
- nginx: ~2,000,000 RPS

These numbers are 28-core Linux with pipelining. On a single-route GET, single client, **no pipelining**, M-series macOS (kqueue, not epoll, fewer kernel optimizations), expect 1/10th to 1/20th. axum on a comparable single-core macOS setup with keepalive lands around 150-300k RPS. Hyper on macOS, single core, keepalive: 200-400k.

So your gap is not "40k vs 7M" — it's "40k vs ~200k", which is **5x**. Where does that 5x live?

1. **Keepalive: ~3-4x.** Close-per-request burns the connection. This is the biggest single lever.
2. **Per-I/O syscall overhead (register/deregister churn): ~1.5x.** Stop re-registering per future.
3. **Tight-poll waker substitute: ~1.1-1.3x.** Real wakers reduce wasted polls.
4. **AST-rewriter yield overhead: unknown, possibly meaningful.** Profile this. If `Thread.yield_now` is in your top-5 hottest functions, you have a runtime architecture problem, not a tuning problem.
5. **Scheduler choice: ~0%.** Work-stealing won't help. Single-handler-per-connection is already on one core.

The fact that rondo-sync (1 thread, blocking) gets 42k while async-multi (4 workers) gets 40k is the smoking gun. The async path is **slower per-thread** because of the register/deregister/poll overhead, and the 4-worker parallelism barely compensates. With keepalive on, async-multi should pull cleanly ahead because the per-request fixed cost drops, and the I/O concurrency starts to matter.

---

## 5. v2 roadmap

Three milestones, ordered by RPS-per-unit-complexity.

### M1 — HTTP/1.1 keepalive in Rondo (highest ROI, lowest runtime risk)

Deliverables:
- `Connection: keep-alive` parsing + emission
- Per-connection request loop in `handle_one`: read → dispatch → write → check `Connection: close` → loop
- Idle timeout (configurable; default 60s)
- `Content-Length` / `Transfer-Encoding: chunked` body framing
- Update bench to reuse connections; verify 3-5x RPS on the same single-route GET

Expected: 40k → 150-200k RPS. Almost entirely a framework-layer change; no runtime changes required.

### M2 — Reactor lifecycle: persistent fd registration + real wakers

Deliverables:
- `AsyncTcpStream`/`AsyncTcpListener` register fd once at construction with `EPOLLET` (Linux) / `EV_CLEAR` (macOS)
- Drop deregisters then closes
- Edge-triggered readiness with retry-on-EAGAIN loop inside read/write
- Per-reactor wake fd: eventfd on Linux, EVFILT_USER on macOS
- `Waker` ABI: `(reactor_ptr, task_slot_id)` → writes wake fd
- Replace AST-rewriter tight poll with park-on-pending, wake-on-ready

Expected: 1.5-2x on top of M1. Higher complexity — touches the runtime C layer. This is also where you fix the "futures aren't really futures, they're poll-until-ready loops" architecture debt.

### M3 — Bounded backpressure + connection state machine

Deliverables:
- Bounded per-worker connection cap (reject with `503` or `accept` queue depth limit)
- Per-connection state machine (HEADERS / BODY / IDLE / CLOSING)
- Configurable per-worker max in-flight requests
- Static keepalive-aware metrics: `connections_active`, `connections_idle`, `requests_per_connection_p99`
- Stack size default → 1 MB, configurable

Expected: small RPS gain (~10-20%) but **eliminates the failure mode where one slow handler kills throughput on one shard**. This is the production-readiness step.

What's **not** on the roadmap: work-stealing, cross-thread task migration, io_uring, a global runtime. Those are v3+ decisions, made *after* you've measured what M1-M3 leave on the table.

---

## Sources

- [Making the Tokio scheduler 10x faster](https://tokio.rs/blog/2019-10-scheduler)
- [tokio multi-thread worker source](https://docs.rs/tokio/latest/src/tokio/runtime/scheduler/multi_thread/worker.rs.html)
- [mio waker source](https://github.com/tokio-rs/mio/blob/master/src/waker.rs)
- [mio epoll selector](https://github.com/tokio-rs/mio/blob/master/src/sys/unix/selector/epoll.rs)
- [tokio AtomicWaker source](https://github.com/tokio-rs/tokio/blob/master/tokio/src/sync/task/atomic_waker.rs)
- [Tokio I/O driver design](https://deepwiki.com/tokio-rs/tokio/3.3-notification-and-signaling)
- [Go netpoll source](https://go.dev/src/runtime/netpoll.go)
- [Go scheduler deep dive](https://nghiant3223.github.io/2025/04/15/go-scheduler.html)
- [Datadog: introducing glommio](https://www.datadoghq.com/blog/engineering/introducing-glommio/)
- [monoio vs tokio/glommio](https://zread.ai/bytedance/monoio/30-comparing-with-tokio-and-glommio)
- [Cloudflare: sad state of Linux socket balancing](https://blog.cloudflare.com/the-sad-state-of-linux-socket-balancing/)
- [Envoy listener connection balancing](https://medium.com/@hnasr/envoys-listener-connection-balancing-e015a9b09a85)
- [Envoy threading model](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/intro/threading_model)
- [Deep dive into the Erlang scheduler](https://blog.appsignal.com/2024/04/23/deep-diving-into-the-erlang-scheduler.html)
- [F5: HTTP keepalive and web performance](https://www.f5.com/company/blog/nginx/http-keepalives-and-web-performance)
- [TechEmpower Round 23 benchmarks](https://www.techempower.com/benchmarks/)
- [io_uring vs epoll for networking — Alibaba](https://www.alibabacloud.com/blog/io-uring-vs--epoll-which-is-better-in-network-programming_599544)
