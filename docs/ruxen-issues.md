# Ruxen compiler issues found while building Rondo

A running log of compiler gaps + parser quirks encountered while
building Rondo (a Sinatra-style web framework on Ruxen). Each entry
has: a minimal reproducer, the fix or current workaround, and
whether the upstream Ruxen repo has it patched.

Some of these I fixed in-place and committed upstream during this
session; the rest are documented workarounds that Rondo applies
today and need real fixes later.

Date: 2026-05-23. Ruxen branch: `v1-missing-features`.

---

## Fixed upstream (this session)

### F1. `def var <method> -> T; … self` returned garbage

**Symptom:** `r = r.with_status(201)` on a builder that returns
`self` from a `def var` method read back `r.status == 4` (or
`6788`, depending on heap reuse) instead of `201`. The mutation
itself was correct; the return value was wrong.

**Root cause:** `mir/lower/expr/assign.rs:42-48` unconditionally
inserted `ruxen_dealloc(R)` before `Assign { dest: R, value: Use(X) }`
when `R` was a heap-owned local being rebound. When `X` came from
a call returning `self`, `X` aliased `R` — the dealloc freed the
very heap object the assign re-read. Use-after-free.

**Fix:** new post-lowering pass
`drops::elide_returns_self_realloc` (ruxen commit `0a12b18`).
Identifies functions returning their first param (`Return(Some(Use(params[0])))`
on every returning block), walks every block looking for the
buggy triple `Call X = F(R,…) ; ruxen_dealloc(R) ; Assign R = X`
where `F` is in the returns-self set, and removes the dealloc.

### F2. Match-arm pattern bindings had `Ty::Infer` instead of payload type

**Symptom:** `match opt; Some(t) -> t.describe(); ...` with
`opt: Option[Thing]` link-failed with
"no runtime symbol for `?T518_describe`". Pattern bindings used in
interpolation printed as integers (`hi 54303703232`).

**Root cause:** `typeck/infer/expr.rs::HirExprKind::Match`
inferred the scrutinee then walked arm bodies without touching
pattern bindings. The bindings allocated DefIds at resolve time
got their types updated nowhere.

**Fix:** new `helpers::propagate_pattern_types` walker that
structurally matches the pattern against the scrutinee's resolved
type and writes each `HirPattern::Binding`'s DefId-type back into
the symbol table BEFORE the arm body is inferred. Handles built-in
`Ty::Option`/`Ty::Result` (peels payload directly) and user-
defined enums (looks up the variant by `variant_idx`, reads
payload tys from `DefKind::EnumVariant.kind`, substitutes generic
params using the scrutinee's `generic_args`). Tuple / Struct /
Or / Ref / Wildcard / Rest also handled. Ruxen commit `9c7028b`.

### F3. Nested multi-statement match arms didn't parse

**Symptom:**

```ruxen
match a
  Some(x) ->
    match b
      Some(y) ->
        let z = x + y
        puts "z=#{z}"
      None -> puts "no b"
    end
  None -> puts "no a"
end
```

produced "expected expression, found Arrow" at the inner `None ->`.

**Root cause:** `parser/methods.rs::parse_body` only terminated
on `End`/`Else`/`Elsif`/`Eof`. Inside a nested match, the inner
arm body greedily consumed the outer arm's pattern as a
statement.

**Fix:** new `parse_match_arm_body` variant called from
`parse_match_arm` that adds "next tokens look like a sibling arm
header" as a termination signal. `looks_like_sibling_match_arm`
peeks up to 16 tokens for `->` at the current bracket depth,
hard-stopping at any statement-starting keyword
(`let`/`var`/`if`/`while`/`for`/`match`/`return`/`break`/
`continue`), newline at depth 0, or terminator. Ruxen commit
`0257989`.

### F4. Option's `Some` variant_idx off-by-one in MIR

**Symptom:** Even with F2 in place, `match Some(s); Some(n) -> "hi #{n}"`
printed `n` as a pointer-valued integer instead of the captured
string. Pattern bindings looked correct in typeck but MIR
allocated the binding's local with `Ty::Int`.

**Root cause:** `mir/lower/match_arms.rs:148` matched on
`variant_idx == 0` for `Some(T)`, but the resolver registers
Option as `None=0, Some=1` (`resolve/stdlib/option.rs:43-65`).
So `Some(x)` matched the `variant_idx == 0` arm and got an empty
`variant_field_types`, which fell back to `Ty::Int`.

**Fix:** flip the variant-idx guard so `Some(T)` matches
`variant_idx == 1`. Result's arms were already correct. Ruxen
commit `80acb14`.

### F5. `AsyncTcpListener.accept` died on `ECONNABORTED`

**Symptom:** rondo's `serve-async` (single-reactor) and
`serve-async-multi` (4 SO_REUSEPORT workers) processes died after
~2.5 s of `wrk` load on macOS. Process exited cleanly via
`Ok(nil)` — the async accept loop hit `Err(_)`, set
`keep_going = false`, and returned. In async-multi the same path
killed one worker per ECONNABORTED, leaking its LISTEN fd in the
kernel and wedging ~25% of subsequent connections via
SO_REUSEPORT routing to the dead listener's accept queue.

**Root cause:** Classic BSD/macOS gotcha (Stevens UNP §15.6) —
after kqueue signals the listener "readable," the head-of-queue
connection can abort (peer RST between SYN/ACK and accept).
Blocking accept hides this; the kernel silently dequeues and
keeps blocking. Non-blocking accept surfaces it as
`errno == ECONNABORTED`.
`library/std/async_net/runtime/async_net.c::ruxen_async_accept_step`
correctly retried on `EINTR` and parked on `EAGAIN/EWOULDBLOCK`
but treated every other errno — including `ECONNABORTED` — as
fatal, bubbling it up to the user's accept loop.

**Fix:** Treat `ECONNABORTED` as a soft retry (loop back inside
the C state machine, don't surface to user code). Matches Go's
`net` package, tokio, and libevent behaviour. Same fix applied
to the sync side
(`library/std/net/runtime/tcp.c::ruxen_tcp_listener_accept`) so
the BSD gotcha is plugged across the whole stack — `EINTR` is
still propagated there so cooperative SIGINT loops keep working.

### F6. `Thread.spawn` was slow on macOS due to default 512KB stack

**Symptom:** rondo's `serve-threaded` mode capped at ~1.6k RPS
regardless of concurrency. Measured per-cycle cost: sync (no
spawn) 32 µs/req, threaded 616 µs/req → `Thread.spawn` added
~584 µs per call on macOS. Profiling showed `pthread_create` was
the bottleneck — only 3-6 worker threads ever alive under a
200-connection load (workers exit nearly as fast as the main
thread can spawn them).

**Root cause:**
`library/std/sync/runtime/thread.c::ruxen_thread_spawn` called
`pthread_create(&tid, NULL, …)` — `NULL` attr means OS default
stack size. macOS default is 512KB; XNU's per-thread setup cost
scales with the configured stack.

**Fix:** Set an explicit default via
`pthread_attr_setstacksize`. Initially 256 KB (16× macOS's
`PTHREAD_STACK_MIN`). Later **rolled back to 512 KB** in
ruxen `1d6e245` — matching macOS's pthread default exactly —
because 256 KB was below the OS default and risky for FFI
handlers whose stack depth gets non-trivial (LLVM-generated
code with large stack frames, deep recursive parsers).
Overridable per-process via `RUXEN_THREAD_STACK_KB` env
(0 = OS default — useful for stack-overflow debugging on
Linux where the OS default is 8 MB).

### F7. `AsyncCloseFuture` only shutdown, never closed → fd leak

**Symptom:** under sustained wrk load, rondo's async listener
exhausted the kernel's file table (`kern.maxfiles` hit on macOS),
wedging the box. Per-connection lsof showed accepted sockets
stuck in TIME_WAIT with the fd still held by the server process.

**Root cause:** `ruxen_async_tcp_stream_shutdown` (called by
`AsyncCloseFuture.poll`) issued `shutdown(SHUT_RDWR)` but relied
on Ruxen's drop elaboration to invoke `AsyncTcpStream`'s
FFI-bound `def drop` for the stream field of the close future.
That drop hook wasn't firing reliably, so every accepted
connection leaked its fd.

**Fix:** `ruxen_async_tcp_stream_shutdown` now does
`shutdown(SHUT_RDWR)` + `close(fd)` + mark the stream closed
inline. The `close` surface is authoritative — by the time
`block_on(stream.close())` returns Ready, the fd is gone. Ruxen
`1b888b8`.

### F8. Per-poll register/deregister churn on AsyncTcpStream

**Symptom:** Rondo's keep-alive bench plateaued at ~52 k RPS;
profiling showed per-request `kevent EV_ADD` + `EV_DELETE` pairs
for the read fd. Each I/O step did register-on-EAGAIN +
deregister-on-Ready, costing 2 extra syscalls per step (6+ per
HTTP request on top of read/write/close itself). Also tripped
W18 — many keep-alive cycles on one connection eventually
destabilised the worker (suspect slot-table accumulation in the
reactor under the churn).

**Fix:** Persistent fd registration. `AsyncTcpStream` registers
its fd ONCE at construction (in `ruxen_async_tcp_stream_from_fd`)
with edge-triggered readiness (`EV_CLEAR` / `EPOLLET`), stores
the slot handle on the stream struct, and deregisters once at
drop. Read/Write futures only consult `check_fired` — they don't
touch the reactor's slot table per poll. Same shape as tokio's
`AsyncFd`. Ruxen `c6f1e2d`. RPS up 9% (51 k → 56 k); W18 crash
disappears.

### F9. `AsyncTcpStream.read_with_timeout` — idle timeout on async reads

**Symptom:** there was no way to bound how long
`block_on(stream.read(N))` waited. A client that opened a
keep-alive connection and never sent data pinned the worker
forever — keep-alive in production needs an idle timeout
(nginx defaults to 75 s, envoy to 60 s).

**Fix:** New future `AsyncReadWithTimeoutFuture` pairs the read
state machine with a per-poll `ruxen_reactor_register_timer(ns)`.
Park waits on EITHER fd-readiness OR timer fire; the next poll
resolves the race deterministically and the loser's slot is
deregistered. Returns `Err(IoError.TimedOut)` on timer win.
Ruxen `8d63e9d`. Rondo uses it with a 60 s default; verified
end-to-end (a silent client is dropped at 3006 ms when
configured for 3 s).

### F10. Real wakers + `Task.spawn_raw`

**Symptom:** `Waker` was a no-op singleton. The only way to
drive a future was `block_on(future)` — no intra-worker
concurrency, every connection blocked the worker until its
keep-alive session ended.

**Fix:** Each per-thread reactor owns a wake fd
(`eventfd(EFD_NONBLOCK)` on Linux, `EVFILT_USER` on macOS)
registered with itself. `Waker.wake()` writes 8 bytes to the
wake fd; the parked thread unparks and re-polls. `Task.spawn_raw`
adds futures to a per-reactor task table; `block_on` becomes a
real scheduler loop driving the entry future + all spawned
tasks. Ruxen `59b8de6`. Single-threaded per reactor (no
cross-thread spawn yet). Verified via a 100-task fixture
(sum 1..100 = 5050). Rondo now uses this in the worker accept
loop: each accepted
connection is moved into an `async_handle_one_task(...)` future and
enqueued with `Task.spawn_raw`, so the worker returns to accepting
while live keep-alive sessions continue on the same reactor.

---

### F11. FFI class `.new` returning `Result` was shadowed by the constructor-returns-Self rule

**Symptom:** `match Regex.new(pat, &flags); Ok(rx) -> rx.is_match(text); …`
failed at codegen with `no runtime symbol for ?T::is_match (mangled
?T<n>_is_match)`. `Regex.new` is an FFI static method declared
`-> Result[Regex, RegexError]`, but the inferencer typed the call as
`Regex`. So the `match` scrutinee was `Regex` (not `Result`), the
`Ok(rx)` binding never resolved to a concrete type, and the method
call on `rx` mangled with an unresolved `?T` receiver. This was the
concrete blocker for per-param regex route constraints
(`segment_satisfies` in `helpers.rx`) and a specific instance of the
W5/W6 family.

**Root cause:** `typeck::method_resolvers::builtin_method_type` had a
catch-all `(Ty::Class { .. }, "new") => Some(ty.clone())` that assumed
**every** `class.new` yields `Self`, overriding the class's own
declared `new` signature.

**Fix:** that arm now consults `lookup_class_method_return(name,
"new")` first and honours a declared `new`'s return type; classes
built via a plain `def init` (no `new` method symbol) still yield
Self. Concurrency types (Mutex / Arc / SharedSync) match their own
`new` arms earlier, so their Send-bound (E1101/E1102) checks are
untouched. Regression spec: `tests/release-e2e/cases/907_regex_match_dispatch.rx`
in the Ruxen tree. Verified: all `ruxen_core` tests green (517+),
rondo `constraints_test` green.

---

## Workarounds in current Rondo (need real upstream fixes)

### W1. Multi-statement match arm bodies need a statement keyword first

**Symptom:** A multi-statement arm body whose first token is an
expression-start (identifier, method call, `if`, assignment)
parses as a SINGLE expression — subsequent lines are not part of
the body.

```ruxen
Some(params) ->
  matched_idx = i         # ← consumed as the whole arm body
  matched_params = params # ← parser thinks this starts the next arm
None -> 0                 # ← "expected expression, found Arrow"
```

**Workaround in Rondo:** lead the body with a `let _foo = …`
binding to force block parsing:

```ruxen
Some(params) ->
  let _captured = params
  matched_idx = i
  matched_params = _captured
```

(see `Rondo.dispatch` in `src/lib.rx` for the live example).

**Proper fix:** `parser::is_expression_start` is consulted in
`parse_match_arm` to choose Expr-vs-Block. The single-vs-block
distinction shouldn't live there — instead, parse arm bodies as
"a block whose terminator is `<next arm header>` OR `end`". The
sibling-arm-header heuristic I added for F3 should drive the
block-mode choice unconditionally; if the heuristic finds a
sibling, that's a block; otherwise we have one or two statements
ending in an expression.

### W2. `if/else` arm body drops subsequent statements

**Symptom:** Same root as W1 — `if` returns true from
`is_expression_start`, so an arm body starting with `if` is parsed
as a single if-expression and any statements after `end` are
mis-attributed.

**Workaround:** Extract the multi-statement body into a free fn,
keep the arm body as a single call.

### W3. `()` doesn't reliably parse as an expression

**Symptom:** Used as a no-op match arm body (`None -> ()`),
the `()` doesn't always parse — Ruxen seems to lack the unit
literal as a standalone expression in some positions.

**Workaround:** Use `0` (or any other expression of compatible
type) for no-op arms. The match's value is discarded by the
caller anyway.

### W4. Closure bodies are single-expression only

**Symptom:** `{ |req: Request| let x = …; let y = …; … }` — only
the first statement parses as the body; the rest is dropped or
reported as syntax error.

**Workaround:** Extract closure bodies into free fns and call
them from the closure:

```ruxen
app.get(&"/hello/:name", { |req: Request|
  Response.text(&hello_body(&req))   # helper does the work
})
```

This is actually fine for Sinatra-style ergonomics — Sinatra
encourages thin routes — but it'd be nicer to support
multi-statement closure bodies via a block delimiter.

### W5. Generic-method dispatch fails when receiver type is `?T`

**Symptom:** `for route in self.routes.iter` followed by
`route.matches(...)` link-fails with "no runtime symbol for
`?T100_matches`". The iterator's element type isn't carried into
the method-call lowering.

**Workaround:** Indexed loop via `routes.get(i)` + pattern-match
on the returned `Option[&Route]`. Combined with F2 (pattern type
propagation), the binding's type resolves cleanly.

```ruxen
let n: Int = self.routes.len() as Int
var i: Int = 0
while i < n
  match self.routes.get(i)
    Some(route) ->
      match route.matches(&req)
        ...
```

**Proper fix:** Monomorphization should propagate the array's
element type into iterator-element method dispatch. Tracked in
project memory as the "generic stdlib types lose T in MIR" gap.

### W6. `Option.unwrap()` / `Option.is_some()` etc. link-fail on generics

**Symptom:** `opt.unwrap()` where `opt: Option[String]` link-fails
with `Option[String]_unwrap` — codegen mangles the parameterised
type name literally and no monomorphized symbol exists.

**Workaround:** Pattern-match exclusively.

```ruxen
let v = match opt
  Some(x) -> x
  None    -> default
end
```

### W7. `BufReader.Tcp.read_line` dotted-class FFI alias doesn't resolve

**Symptom:** Calling `BufReader.Tcp.new(stream).read_line` link-
fails with undefined symbols `BufReader.Tcp_read_line`,
`BufReader.Tcp_into_inner`, `BufWriter.Tcp_flush`. The FFI alias
clause in stdlib (`def read_line as "ruxen_bufreader_read_line"`)
should rewrite the mangled callee, but the dotted-module class
name (`BufReader.Tcp_…`) escapes the rewriter.

**Workaround in Rondo:** Don't use `BufReader.Tcp`. The TCP
server in `Rondo.listen` reads NOTHING from incoming connections
(writes a canned response and closes). Request parsing over a
real socket lands once this alias gap closes.

### W8. No bytes→String conversion in stdlib

**Symptom:** `TcpStream.read(&var Array[Int])` reads bytes into
an Array[Int], but there's no `String.from_bytes(&Array[Int])`
or similar to turn those bytes into a String we can feed to
`Request.parse`.

**Workaround in Rondo:** Same as W7 — skip request parsing over
sockets for now.

**Proper fix:** Either expose `ruxen_string_from_utf8(bytes)`
through stdlib or add a `TcpStream.read_to_string` that delegates
to the existing `ruxen_stream_read_to_string` runtime helper
(currently only used by `File` and `Stdin`).

### W9. `derive Clone` (and other derive keywords) silently ignored

**Symptom:** `class Request; derive Clone; …` errors with
"expected Colon, found TypeIdentifier" — the parser doesn't
recognise `derive` as a body keyword for user classes. Looking
at `parser/classes.rs`, `derive_traits` is always initialised to
`Vec::new()` and never populated.

**Workaround in Rondo:** Avoid needing Clone — Rondo's
before-hooks take `&Request` not `Request`, so no copy is needed.

### W10. Forward references in helper signatures fail

**Symptom:** A free `def foo(x: &Request)` declared BEFORE
`class Request` errors with "undefined type `Request`". Forward
references work in fn bodies (resolved at the second pass) but
not in parameter / return-type positions.

**Workaround in Rondo:** All helper fns whose signatures
reference user types (`Request`, `Response`, `Route`, `Rondo`,
`TcpStream`) live at the END of `src/lib.rx`, after the class
definitions.

### W11. `String.split("/")` returns SplitIter on `Str` receivers

**Symptom:** Calling `.split(&"/")` on a string literal directly
gives back `Ty::Class { name: "SplitIter" }` instead of
`Array[String]` — and `SplitIter` doesn't expose `.get(i)` or
`.len()`, so indexed iteration link-fails.

**Workaround:** Make sure the receiver is an owned `String`, not
a `&str`. In Rondo's case, fields are `String`-typed and helper
args bind via `&String`, so this is fine on the framework's hot
paths.

### W12. String-literal-to-String coercion in Option/Result payloads — **RESOLVED by the literal model (2026-06-11), see M1**

**Symptom (historical):** `Some("ada")` where the declared return type is
`Option[String]` failed typeck with
"expected Option[String], found Option[&str]". String literals
defaulted to `&str` and weren't auto-coerced into the payload type.

**Historical workaround:** `Some(String.from(&"ada"))` — explicit
construction.

**Now:** under the bare-literal model (M1) a bare `"ada"` IS an owned
`String`, so `Some("ada")` typechecks directly. The residual coercion gap
is narrower and lives only in tuple-element position — see W21.

### W13. `pub` keyword not supported

**Symptom:** `pub class Foo` / `pub def bar` errors with
"expected top-level declaration, found Identifier(\"pub\")".
Visibility is implicit in Ruxen; there's no equivalent today.

**Workaround in Rondo:** Just omit `pub`. All items in
`lib.rx` are accessible to depending packages by default.

### W14. Path-dep symbols are flat-merged, not module-namespaced

**Symptom:** `use rondo.Rondo` from a dependent package fails
with "unknown module 'rondo'". The dep's source IS being merged
into the user's compile (via the `compile_project` flat-merge
fix Codex committed earlier today), but the symbols end up at
the top level rather than namespaced under `rondo`.

**Workaround in Rondo:** No `use` statements in `rondo-smoke`;
`Rondo`, `Request`, `Response` are available at the top level.

**Proper fix:** Wrap each dep's program in a module DefId during
resolve so `use rondo.X` resolves through `Module(rondo) →
Class(X)`.

### W15. `Thread.spawn` closure crashes when worker calls `block_on` and captures references — **FIXED in ruxen `a182502`**

The async-multi listener with 4 workers now boots cleanly and
serves load (verified by Rondo's bench). Description preserved
below for archeology.

**Symptom (historical):** the multi-worker async serving model
(`Rondo.listen_async_multi`) segfaulted the spawned thread
**before any line of the worker body ran**. Verified by
adding `puts "worker: entering"` as the first statement of
the captured closure body — no output is printed before
the SIGSEGV.

Setup that crashes:

```ruxen
def async_worker(app: &Rondo, addr: &String) -> Int
  puts "worker: entering"   # never prints
  let bound = block_on(AsyncTcpListener.bind(addr))
  ...
end

def spawn_async_workers(app: &Rondo, addr: &String, workers: Int)
  let _h = Thread.spawn({ || async_worker(app, addr) })   # ← segfault
  ...
end
```

Reduction that **works**:
- `Thread.spawn({ || diag_worker() })` where `diag_worker` takes
  no args (no captures) and calls `block_on(AsyncTcpListener.bind(...))`
  internally — the worker runs and prints, the join returns 7.
- `Thread.spawn({ || run_request(app, stream) })` where `app: &Rondo`
  and `stream: TcpStream` (TcpStream captured by move) — the
  threaded-per-connection variant (`listen_threaded`) works under
  thousands of `ab` requests without failure.
- The same `async_worker(app, addr)` called directly on the main
  thread serves traffic correctly — confirms the worker body itself
  is fine.

So the crashing combination is: **`Thread.spawn` closure that
captures TWO non-Send-moved values (any of: `&Rondo`, `&String`,
owned `String` via `addr.clone`) AND whose body calls `block_on(...)`.**
A single-capture closure of a moved value works. A capture-less
block_on works. The interaction of capture + block_on is the gap.

The Ruxen side calls this out in
`docs/rondo_v1_blockers.md` cross-reference for B1, citing
`project_ruxen_task_spawn_ownership_gap.md` ("Task.spawn already
has a drop-elaboration gap; the same shape on Thread.spawn is
the likely failure mode") — this is exactly that mode.

**Workaround in current Rondo:** `Rondo.listen_async_multi` is
shipped but its body falls back to running the async accept
loop on the main thread (equivalent to `listen_async`) and
prints a warning if the caller asked for >1 worker. The
`SO_REUSEPORT` runtime patch (ruxen `async_net.c`,
docs/rondo_v1_blockers.md §B5) is in place and verified by
the `reuseport-check` mode of rondo-smoke (`bind` from two
threads on the same port both succeed), so the only thing
blocking real multi-core serving is this Thread.spawn capture
gap.

**Acceptance test for the fix:** the ruxen-side fixture
should match this exact shape — capture `&Rondo` and `&String`
(or owned `String`) in a `Thread.spawn` closure whose body
calls `block_on(...)`, and ensure the worker's first `puts`
fires.

### W16. Tuple returns of FFI-owning classes double-drop

**Symptom:** a helper that takes ownership of a `TcpStream` and
returns `(TcpStream, Bool)` SIGSEGVs the worker on the SECOND
keepalive iteration. Both the tuple's stored stream and the
caller's reassigned binding try to drop; the second close
hits a freed fd.

**Workaround:** use `Option[T]` instead of tuples for any
function that hands a stream back to its caller. `Some(stream)`
correctly moves the payload out through `unwrap`. This is why
the early sync-server keep-alive attempt was reverted to
close-per-request — see `handle_one` commit history.

**Acceptance test for the fix:** drop-elaboration should treat
tuple-extract (`result.0`) the same as `Option.unwrap` for
move-semantics.

### W17. FFI-bound `def drop` doesn't always fire for class fields of a future

**Symptom:** `AsyncCloseFuture.stream` had a
`def drop as "ruxen_async_tcp_stream_drop"` but the field's
drop wasn't invoked when the close future itself dropped,
leaking the fd. The shape is: future class with a class-typed
field whose class has a `lib`-bound drop hook.

**Workaround in current Rondo:** the `close` C-side method does
the fd close inline (not just `shutdown`) so the stream's own
drop becomes a no-op safety net. See ruxen `1b888b8`.

**Acceptance test for the fix:** drop elaboration should walk
class field types and emit FFI-aliased drop calls in
declaration order on scope exit — for future classes, this
fires when `block_on` returns the future's Output.

### W18. Many keep-alive iterations on a single async TCP connection eventually crash the worker — **FIXED in ruxen `c6f1e2d`**

The persistent-fd-registration shape (F8) eliminates the
per-cycle reactor slot churn that caused this. Rondo's
keep-alive bench now sustains unbounded iterations per
connection (verified across c=50/200/800 and 800+ requests per
connection without incident).

### W19. `.await` inside `while` loop body — E1115 — **FIXED in Ruxen `115a56f`**

**Symptom:** the ruxen compiler used to emit **E1115** when an
`async def` body contained a `while` loop whose body performed a
`.await`. Async-lowering's state-machine codegen did not model
the loop-back edge — every `.await` became a state transition, but
the body of a loop generated an edge back to the loop-condition
state that the pass did not emit.

**Why this matters for Rondo:** the natural shape for
intra-worker concurrency is

```ruxen
async def serve(app: &Rondo, l: AsyncTcpListener)
  while keep_going
    let pair = l.accept.await
    match pair
      Ok(p) -> Task.spawn_raw(handle_one_async(app, p.0))
      Err(_) -> keep_going = false
    end
  end
end
```

**Current Rondo shape:** the runtime piece (`Task.spawn_raw` +
real wakers, F10) is in place, and Rondo now uses it in the
accept loop. The accepted stream is moved into a spawned async
connection task. The connection task follows the supported
multi-await loop form: direct await-let phases in the loop body,
with synchronous response construction between phases.

**Historical workaround:** unroll `.await` calls (each at the top
level of the async fn, not inside any loop). This worked for
fixed-count spawns but was not viable for accept loops.

---

### W20. `ruxen test` v1 is single-file — can't link the library under test

**Symptom:** `ruxen test` discovers `tests/**.rx`, but a test file
cannot reference Rondo's own classes. Both forms fail:

```rx
# tests/dispatch_test.rx
use rondo.Rondo            # error: expected expression, found Use
Tester.describe("x") do |t: &var Tester|
  t.it("y") do
    let r = Response.text(&"hi")   # error: undefined variable `Response`
    t.expect(r.status).to_eq(200)
  end
end
```

**Root cause** (read from the toolchain source):

1. The runner wraps the **entire** test-file body inside a
   synthesised `def main` (`src/ruxenc/src/test_runner.rs`,
   `synthesise_wrapper`): `def main … <your file verbatim> … r.execute end`.
   So a test file may contain **only statements** — a top-level
   `def`, `class`, or `use` lands inside `main` and is a parse error.
   You cannot define handler helper fns, and you cannot `use` anything.
2. Each wrapper compiles via the **single-file** `ruxenc` driver, not
   project-aware `ruxen build`. `src/ruxenc/src/compile.rs` is explicit:
   *"today ruxenc is single-file, so the dep list is empty."* The
   project's own `src/*.rx` and any dependency are **not** flat-merged
   in, so none of Rondo's symbols exist in the test binary. `tests/support/`
   is only skipped from discovery — it is not linked either.

**Consequence:** the merged-source workaround (concatenate `src/*.rx`
ahead of the test) also fails — those `def`/`class`/doc-comment lines
end up nested inside the synthesised `main`. In v1 only std-prelude
symbols and locally-`let`/`var`-bound values are reachable, so the
framework can validate self-contained logic but **cannot exercise
Rondo's public API** (`Rondo.dispatch`, `Request.parse`, `Response.*`).

**Workaround in use:** the dispatch seam stays covered by the
in-process `Rondo.dispatch(req) -> Response` checks in
`rondo-smoke`'s `main` (the `✓`/`✗` assertions), which compile through
`ruxen build` and therefore see the whole library. Migrate those into
`ruxen test` files once the runner gains multi-file / dependency
linking (tracked alongside W14's flat-merge resolver work).

**Update (2026-06-11):** Q16 is fixed upstream — `ruxen test` now links
the project library, and all 10 `tests/**.rx` files compile, link, and run
(72 examples, all green). The single-file-runner limitation in W20 no
longer applies; the public API (`Rondo.dispatch`, `Request.parse`,
`Response.*`) IS exercised through `ruxen test` today. (`rondo-smoke` is
also no longer present on disk — the docs above and `CLAUDE.md` still
reference it as the integration harness; treat those references as stale
and run `ruxen test` directly.) Surfacing this also re-exposed W6 — see M1
below.

### W21. Bare string literal not coerced to `String` in tuple-element position

**Symptom:** under the bare-literal model (M1), a bare `""` literal placed
as a tuple element whose expected element type is `String` is NOT coerced —
it stays `&str`, so the tuple's type is `(&str, Bool)` instead of the
expected `(String, Bool)`.

Minimal repro:

```ruxen
def make_pair(b: Bool) -> (String, Bool)
  if b
    ("", false)        # error: expected (String, Bool), found (&str, Bool)
  else
    ("x", true)
  end
end
```

The coercion that works in `let` / field / param / `Some(...)` positions
does not propagate into tuple elements.

**Workaround in Rondo:** spell the element `"".to_string()` (an owned
`String`). `async_cycle_wire` in `src/async_server.rx` uses
`("".to_string(), false)` for its two empty-wire returns. `.clone` on a
bare literal link-fails here, and `String.from` is being removed, so
`.to_string()` is the forward-compatible spelling.

**For the coordinator's ruxen ledger:** this is a genuine new compiler gap
(literal→String coercion missing in tuple-element position). Repro above;
severity low (one-line workaround). Q-number to be assigned centrally.

---

## Migration log

### M1. String-literal model + `Map`→`Hash` + W6 resurfacing (2026-06-11)

The installed toolchain (`local-1b45eb1`) landed several breaking changes at
once. Rondo migrated to all of them on branch `feat/string-literal-model`:

1. **Bare literals are owned Strings; `String.from` deleted.** A bare
   `"literal"` is now an owned `String` in every position (params, fields,
   `Err(...)` into `Result[T, String]`, collections, returns, let-bindings).
   `String.from` is being removed from the stdlib; `.clone` (same C symbol)
   is the spelling for a runtime borrow→owned copy. Rondo swept all 133
   `String.from(&"literal")` sites (6 in `src/`, 127 in `tests/`) down to the
   bare literal — every site was the borrowed-literal shape, so no `.clone`
   rewrites were needed. `String.from_bytes` (the bytes→String conversion
   behind W8) is a *different* method and is preserved. Residual coercion gap:
   W21 (tuple elements). This obsoletes the W12 workaround.

2. **`Map` renamed to `Hash` (stdlib rename, NOT a compiler bug).** The
   prelude map type is now `Hash[K, V]` (same `insert`/`get`/`keys`/`len`
   API). `Hash` is live across the whole Ruxen stack (quiver/canvas/stdlib);
   rondo was the only repo still on `Map` and so failed `ruxen build` with 34
   `undefined type Map` errors before any change. Renamed all 41 `Map` sites
   (33 `src/`, 8 `tests/`) to `Hash`. No Q-number — this is a rename rondo
   missed, reconciled centrally.

3. **W6 re-exposed by the Q16 fix.** With `ruxen test` now linking the
   library, four `.is_some()`/`.unwrap()` calls on parameterised `Option`
   types (`Option[&Route]_is_some`, `Option[Hash[String,String]]_is_some`,
   etc.) link-failed — the W6 family. `ruxen build` had hidden these because
   the library link didn't force those monomorphisations. Converted all four
   (`rondo.rx` ×2, `router.rx` ×1, `request.rx` ×1) to the codebase's
   documented W6 `match … Some(x) -> … nil -> 0` form.

End state: `ruxen build` green; `ruxen test` 72 passed / 0 failed / 0 pending.

---

## Summary

Rondo's v0.4 surface (route DSL with five HTTP verbs, path-param
extraction, query-string parsing, request parser, response
serializer, TCP listener+accept+write, before-hooks for
short-circuit auth/logging) **works today** with these
workarounds. Real production-grade serving (request parsing over
a socket, async I/O, halt sentinel, after-hooks) is gated on the
remaining items above — primarily W5 (generic-method dispatch),
W6 (Option.unwrap on generics), W7 (dotted-class FFI alias),
and W8 (bytes→String).

The four upstream fixes I committed (F1-F4) are infrastructure
that benefits every external Ruxen project, not just Rondo —
each was already a footgun for any user who hit the right shape.
