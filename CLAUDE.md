# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commit conventions

- **Do not add `Co-Authored-By: Claude` (or any AI co-author) trailer to commits.** Commits in this repo carry only the human author's identity. This overrides the global rule in `~/.claude/CLAUDE.md`.
- Don't add `🤖 Generated with [Claude Code]` blurbs to PR bodies for this repo either.

## What this is

Rondo is a Sinatra-style web framework written in **Ruxen** (toolchain at `~/.ruxen/bin/ruxen`). The framework code is split across `src/*.rx` files; Ruxen resolves classes and free fns across siblings in the same package.

The mental model is `A (route) → B (handler) → A`. Routes are the chorus, handlers are the verses.

## Common commands

```bash
ruxen build                # compile the library (debug → target/debug/)
ruxen check                # typecheck without codegen — fastest feedback loop
ruxen build --release      # release build
ruxen clean                # wipe target/
ruxen fmt                  # format .rx files in-place
ruxen explain E####        # explain a compiler error code
ruxen test                 # discover + run tests/**.rx (RSpec-shaped Tester DSL)
ruxen upgrade --from-source /Users/hassan/.projects/ruxen   # rebuild + install the toolchain after stdlib changes
```

To rebuild Ruxen itself after touching `~/.projects/ruxen`, use `ruxen upgrade --from-source <path>` (`ruxen update` is for project dependencies, not the toolchain). Stdlib changes embedded via `include_str!` only land in the toolchain after a fresh build.

`ruxen test` exists and runs (RSpec-shaped: `tests/**.rx` with `Tester.describe`/`t.it`/`t.expect`), **but its v1 runner is single-file** — it wraps each test file's body in a synthesised `def main` and compiles it with the single-file `ruxenc` driver, so a test file may contain only statements (no top-level `def`/`class`/`use`) and **cannot link the rondo library or any dependency**. That makes `Rondo`/`Request`/`Response` unreachable from a test, so the public API can't be exercised via `ruxen test` yet. See `docs/ruxen-issues.md` W20.

Until the runner gains multi-file/dependency linking, the framework is exercised end-to-end through the sibling `~/.projects/rondo-smoke` crate (which builds via `ruxen build` and so sees the whole library): it serves on `127.0.0.1:8421`, and `Rondo.dispatch(req) -> Response` is the pure, socket-free seam its `main` asserts against (the `✓`/`✗` checks).

## Architecture

Thread-per-core async event loop:

- The server spawns N OS threads (N = `app.workers(N)`), each with its own async reactor, all bound to the same port via SO_REUSEPORT. Kernel hashes incoming connections across the N bindings.
- HTTP/1.1 keep-alive by default — one TCP connection serves many requests.
- `Task.spawn_raw` per accepted connection inside each worker reactor, so multiple keep-alive connections make progress concurrently within one worker.
- Same shape as nginx / envoy / monoio / seastar (the production thread-per-core consensus).

### Public API surface

The app is configured by a fluent DSL on the `Rondo` instance, then started with `run()` or `listen(...)`:

- **Routing:** `get / post / put / delete / patch (path, handler)`. `:name` path segments become params (`req.param(&"name")`).
- **Hooks:** `before(fn(&Request) -> Option[Response])` (short-circuits if it returns `Some`), `after(fn(&Request, Response) -> Response)`, `on_error(status, fn(&Request, Response) -> Response)`.
- **CORS middleware:** `cors()` enables it; `cors_origin / cors_methods / cors_headers / cors_max_age / cors_credentials`, or `cors_policy(...)` to set everything at once. Any cors setter implies `cors_enabled = true`. See dispatch wiring below.
- **Server config:** `host(&str)`, `port(Int)`, `workers(Int)`, `max_connections(Int)` (0 = unlimited; enforced as a bounded active-connection cap in `async_server.rx` via `try_acquire_connection` / `release_connection`).
- **Entry points:** `run()` uses the configured host/port/workers. `listen()` has overloads — `listen()`, `listen(port)`, `listen(&addr_or_host)`, `listen(&addr, workers)` — that override config for that call. All return `Result[(), String]`.

### Package layout

The framework is split across files; Ruxen resolves names across siblings in the same package (top-to-bottom within one file still matters for W10 forward refs):

```
src/
  lib.rx          — package doc, module layout
  helpers.rx      — pure free helpers, primitive-only signatures
                     (segment_at, parse_query_into, parse_header_into,
                      parse_cookies_into, rebuild_body, reason_phrase,
                      extract_method, head_only_wire)
  request.rx      — Request class + Request.parse
  response.rx     — Response class + builders + to_http serializer
  route.rx        — Route class + matches()
  rondo.rx        — Rondo class — registration DSL (get/post/put/
                     delete/patch), hooks (before/after/on_error),
                     CORS middleware (cors/cors_*), server config
                     (host/port/workers/max_connections + the
                     connection-cap acquire/release), dispatch(req),
                     and the run() / overloaded listen() entry points
  router.rx       — late-binding helpers (signatures reference user
                     types): split_path_query_into,
                     invoke_route_handler, find_get_route_for,
                     try_static_or_404, should_keep_alive
  async_server.rx — async TCP server: per-worker reactor +
                     keep-alive request loop. `async_handle_one`
                     loops `async_serve_one_cycle` until the client
                     sends `Connection: close` or the peer closes.
```

The `Rondo.dispatch` pipeline in `rondo.rx` is the single most important piece — most behaviour questions reduce to "what happens at step N of `dispatch`?" The core steps: match → merge params → before-hooks → handler/static/404 → after-hooks → error-handler substitution → return. CORS is woven into this when `cors_enabled`: an `OPTIONS` request short-circuits to a preflight response, and every returned response (handler result, before-hook short-circuit, final) is passed through `cors_response` so the headers are applied uniformly.

### Current bench shape

Apple M4, macOS, single-route GET `/`, 4-worker `rondo-smoke serve` (release), wrk against `127.0.0.1:8421` (measured 2026-06-01, post-`Task.spawn_raw`):

| Load            | RPS      | p50      | p99     | Errors |
|-----------------|----------|----------|---------|--------|
| 4t / 200c / 15s | ~120–124 k | 1.7–2.9 ms | ~3.4 ms | 0 |
| 4t / 50c / 10s  | ~114 k   | 620 µs   | 778 µs  | 0 |

Keep-alive on; 60 s idle timeout per connection; no per-request handshake or close. Each request is read-with-timeout + dispatch + write; the close future only fires when the client requests it or idle fires. To reproduce: `ruxen build --release` in `rondo-smoke`, `./target/release/rondo-smoke serve`, then `wrk -t4 -c200 -d15s --latency http://127.0.0.1:8421/`.

Reactor uses persistent fd registration with edge-triggered readiness (`EV_CLEAR` / `EPOLLET`) — the stream registers once at accept and deregisters at drop, so per-request register/deregister syscalls are gone. This is the tokio `AsyncFd` shape.

`AsyncTcpStream.read_with_timeout(max_bytes, timeout_ns)` races the read against a per-poll timer. After `timeout_ns` of no readiness, it returns `Err(IoError.TimedOut)`. Per-read timer registration costs ~1-2 µs of overhead per cycle.

`Task.spawn_raw` per accepted connection inside each worker reactor lets multiple keep-alive connections make progress within one worker — this is what lifts throughput to the ~120 k ceiling above (the pre-`Task.spawn_raw` shape ran one connection at a time per worker and topped out far lower). Latency is sub-ms at moderate concurrency and rises into the low-ms range at c=200 as requests queue across the spawned tasks. Re-benchmark on the target hardware before quoting a new ceiling — these numbers are M4-specific.

Remaining framework and runtime follow-up work is tracked in
`docs/remaining-tasks.md`.

## Ruxen compiler workarounds — read before editing

The codebase is shaped by the Ruxen compiler's current limitations. **`docs/ruxen-issues.md` is required reading before any non-trivial edit** — it has reproducers and fixes for each. Recurring constraints you will hit:

- **W1 — Multi-statement match arm bodies must start with a statement keyword.** If you want >1 statement in an arm, the first line must be `let _foo = …` (or `var` / `if` / `while` / etc). Otherwise the parser treats the body as a single expression and mis-attributes the rest. See `Rondo.dispatch` for the canonical `let _captured = params` pattern.
- **W2 — Same problem with `if/else` arm bodies.** Extract multi-statement bodies into free fns.
- **W3 — Don't use `()` as an expression.** Use `0` for no-op match arms whose value is discarded.
- **W4 — Closure bodies are single-expression only.** Handlers passed to `app.get(...)` etc. must be one expression — extract real work into free fns (`hello_body(&req)`) and call them from the closure. This is fine ergonomically and matches Sinatra's "thin routes" style.
- **W5 — Generic-method dispatch fails on iterator elements.** Don't use `for x in array.iter` followed by method calls on `x`. Use indexed `array.get(i)` with a `match Some(x) -> … None -> 0`. Every loop over `self.routes`, `self.before_hooks`, etc. follows this pattern.
- **W6 — `Option.unwrap()` / `is_some()` link-fail on parameterized types.** Pattern-match instead: `match opt; Some(x) -> x; nil -> default; end`. (Now that `ruxen test` links the library — Q16 — these link-fail even where they used to compile under `ruxen build`; the whole codebase is on the match form. See migration log M1 in `docs/ruxen-issues.md`.)
- **W10 — Forward references fail in parameter/return-type positions.** Put helpers whose signatures mention user classes at the bottom of the file.
- **W12 — RESOLVED by the literal model.** A bare `"literal"` is now an owned `String` in every position, so `Some("ada")` typechecks directly — no `String.from`. The residual coercion gap is W21 (tuple-element position only); spell those `"".to_string()`. See `docs/ruxen-issues.md` M1.
- **W13 — No `pub` keyword.** Visibility is implicit; everything is reachable from depending packages.

Workarounds W7 and W8 (dotted-class FFI aliases, no bytes→String) are documented but partially gated — confirm the current Ruxen toolchain has those before relying on them.

- **W16 — Tuple returns of FFI-owning classes double-drop.** A helper that takes ownership of a `TcpStream` and returns `(TcpStream, Bool)` leaks ownership: both the tuple's stored stream and the caller's reassigned binding try to drop, double-closing the fd and SIGSEGV'ing on the second iteration. Use `Option[T]` instead (`Some(stream)` keeps ownership transfer clean through unwrap). Documented in the `handle_one` history; this is why the sync-server keepalive path was reverted to close-per-request.
- **W17 — FFI-bound `def drop` doesn't always fire for class fields of a future.** `AsyncCloseFuture.stream` had a `def drop as "ruxen_async_tcp_stream_drop"` but the field's drop wasn't invoked when the future itself dropped, leaking the fd. Workaround: make the explicit `close` method do the fd close, not just `shutdown`. Fixed in ruxen `1b888b8`.
- **W18 — Many keep-alive iterations on a single async TCP connection eventually crash the worker.** Symptom: a connection that serves many hundreds of cycles destabilises the server process. Root cause was per-cycle kqueue/epoll register-deregister churn accumulating reactor-slot state. **Fixed in ruxen `c6f1e2d`** (persistent fd registration on AsyncTcpStream / Listener — registration lives for the stream's lifetime, edge-triggered). The per-connection request cap workaround in `async_handle_one` was removed in rondo after this landed.

When you encounter a *new* compiler quirk, add an entry to `docs/ruxen-issues.md` (W-series for workarounds in code, F-series for fixes committed upstream).

## Conventions specific to this codebase

- **`&String` for read-only string params, `&var T` for mutable refs.** Map iteration uses `let ks = m.keys(); ks.get(i)` because `for k in m.keys()` hits W5.
- **Header names are lowercased on insertion** in both `Request.parse` and `Response.with_header` — this matches Sinatra's case-insensitive lookup and is what HTTP/2 mandates on the wire. Don't break this invariant.
- **`with_cookie` stores under key `set-cookie:<name>`** so multiple cookies don't overwrite each other in the headers Map. `Response.to_http` strips that prefix back out on emission.
- **HEAD requests fall back to the matching GET route**, then the response body is stripped at the wire layer in `handle_one` so `Content-Length` still reflects the GET-shape byte count (RFC 9110 §9.3.2).
- **Static-file fallback rejects `..` segments** in `try_static_or_404` to prevent path traversal — preserve this if you touch the static path.
- **Empty `public_folder` disables static serving.** Don't replace the check with a `nil` sentinel.
