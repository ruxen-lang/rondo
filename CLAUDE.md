# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Rondo is a Sinatra-style web framework written in **Riven** (toolchain at `~/.riven/bin/riven`). The framework code is split across `src/*.rvn` files; Riven resolves classes and free fns across siblings in the same package.

The mental model is `A (route) → B (handler) → A`. Routes are the chorus, handlers are the verses.

## Common commands

```bash
riven build                # compile the library (debug → target/debug/)
riven check                # typecheck without codegen — fastest feedback loop
riven build --release      # release build
riven clean                # wipe target/
riven fmt                  # format .rvn files in-place
riven explain E####        # explain a compiler error code
riven update --from-source /Users/hassan/.projects/riven   # rebuild + install the toolchain after stdlib changes
```

To rebuild Riven itself after touching `~/.projects/riven`, use `riven update --from-source <path>`. Stdlib changes embedded via `include_str!` only land in the toolchain after a fresh build.

There is no `cargo test`-equivalent yet (`riven test` is a placeholder). The framework is exercised end-to-end through the sibling `~/.projects/rondo-smoke` crate, which serves on `127.0.0.1:8421` via `app.listen(addr, workers)`. `Rondo.dispatch(req) -> Response` remains the pure, socket-free seam for in-process tests inside rondo-smoke's `main`.

## Architecture

Thread-per-core async event loop:

- `app.listen(addr, N)` spawns N OS threads, each with its own async reactor, all bound to the same port via SO_REUSEPORT. Kernel hashes incoming connections across the N bindings.
- HTTP/1.1 keep-alive by default — one TCP connection serves many requests.
- Same shape as nginx / envoy / monoio / seastar (the production thread-per-core consensus).

### Package layout

The framework is split across files; Riven resolves names across siblings in the same package (top-to-bottom within one file still matters for W10 forward refs):

```
src/
  lib.rvn          — package doc, module layout
  helpers.rvn      — pure free helpers, primitive-only signatures
                     (segment_at, parse_query_into, parse_header_into,
                      parse_cookies_into, rebuild_body, reason_phrase,
                      extract_method, head_only_wire)
  request.rvn      — Request class + Request.parse
  response.rvn     — Response class + builders + to_http serializer
  route.rvn        — Route class + matches()
  rondo.rvn        — Rondo class — registration DSL (get/post/put/
                     delete/patch), hooks (before/after/on_error),
                     dispatch(req), and the public listen(addr, N)
                     entry point
  router.rvn       — late-binding helpers (signatures reference user
                     types): split_path_query_into,
                     invoke_route_handler, find_get_route_for,
                     try_static_or_404, should_keep_alive
  async_server.rvn — async TCP server: per-worker reactor +
                     keep-alive request loop. `async_handle_one`
                     loops `async_serve_one_cycle` until the client
                     sends `Connection: close` or the peer closes.
```

The `Rondo.dispatch` pipeline in `rondo.rvn` is the single most important piece — most behaviour questions reduce to "what happens at step N of `dispatch`?" The 7 steps: match → merge params → before-hooks → handler/static/404 → after-hooks → error-handler substitution → return.

### Current bench shape

On M-series macOS, single-route GET, 4-worker `listen("127.0.0.1:8421", 4)`, wrk 4t/200c/15s: **~56 k RPS, 17 µs p50, 25 µs p99, zero timeouts.** Keep-alive on; no per-request handshake or close. Each request is read + dispatch + write; the close future only fires when the client requests it.

Reactor uses persistent fd registration with edge-triggered readiness (`EV_CLEAR` / `EPOLLET`) — the stream registers once at accept and deregisters at drop, so per-request register/deregister syscalls are gone. This is the tokio `AsyncFd` shape.

The bench is now CPU-bound on per-request work (parsing, dispatch, serialization), not on connection management. Stable across c=50 to c=800 connections — adding load doesn't change the ceiling because each worker runs one task at a time. The remaining lever for higher RPS is `task::spawn` per connection (multiple in-flight requests per worker); that's a v2 runtime feature. See `docs/research/async-threading-best-practices.md` for the broader v2 roadmap (real wakers, etc.).

## Riven compiler workarounds — read before editing

The codebase is shaped by the Riven compiler's current limitations. **`docs/riven-issues.md` is required reading before any non-trivial edit** — it has reproducers and fixes for each. Recurring constraints you will hit:

- **W1 — Multi-statement match arm bodies must start with a statement keyword.** If you want >1 statement in an arm, the first line must be `let _foo = …` (or `var` / `if` / `while` / etc). Otherwise the parser treats the body as a single expression and mis-attributes the rest. See `Rondo.dispatch` for the canonical `let _captured = params` pattern.
- **W2 — Same problem with `if/else` arm bodies.** Extract multi-statement bodies into free fns.
- **W3 — Don't use `()` as an expression.** Use `0` for no-op match arms whose value is discarded.
- **W4 — Closure bodies are single-expression only.** Handlers passed to `app.get(...)` etc. must be one expression — extract real work into free fns (`hello_body(&req)`) and call them from the closure. This is fine ergonomically and matches Sinatra's "thin routes" style.
- **W5 — Generic-method dispatch fails on iterator elements.** Don't use `for x in array.iter` followed by method calls on `x`. Use indexed `array.get(i)` with a `match Some(x) -> … None -> 0`. Every loop over `self.routes`, `self.before_hooks`, etc. follows this pattern.
- **W6 — `Option.unwrap()` / `is_some()` link-fail on parameterized types.** Pattern-match instead: `match opt; Some(x) -> x; None -> default; end`. (A few `.is_some() / .unwrap()` calls in the file currently work because they're on concrete `Option[Request]` etc. — but prefer the match form for safety.)
- **W10 — Forward references fail in parameter/return-type positions.** Put helpers whose signatures mention user classes at the bottom of the file.
- **W12 — String literals are `&str`, not `String`.** When returning `Option[String]`, write `Some(String.from(&"ada"))`, not `Some("ada")`.
- **W13 — No `pub` keyword.** Visibility is implicit; everything is reachable from depending packages.

Workarounds W7 and W8 (dotted-class FFI aliases, no bytes→String) are documented but partially gated — confirm the current Riven toolchain has those before relying on them.

- **W16 — Tuple returns of FFI-owning classes double-drop.** A helper that takes ownership of a `TcpStream` and returns `(TcpStream, Bool)` leaks ownership: both the tuple's stored stream and the caller's reassigned binding try to drop, double-closing the fd and SIGSEGV'ing on the second iteration. Use `Option[T]` instead (`Some(stream)` keeps ownership transfer clean through unwrap). Documented in the `handle_one` history; this is why the sync-server keepalive path was reverted to close-per-request.
- **W17 — FFI-bound `def drop` doesn't always fire for class fields of a future.** `AsyncCloseFuture.stream` had a `def drop as "riven_async_tcp_stream_drop"` but the field's drop wasn't invoked when the future itself dropped, leaking the fd. Workaround: make the explicit `close` method do the fd close, not just `shutdown`. Fixed in riven `1b888b8`.
- **W18 — Many keep-alive iterations on a single async TCP connection eventually crash the worker.** Symptom: a connection that serves many hundreds of cycles destabilises the server process. Root cause was per-cycle kqueue/epoll register-deregister churn accumulating reactor-slot state. **Fixed in riven `c6f1e2d`** (persistent fd registration on AsyncTcpStream / Listener — registration lives for the stream's lifetime, edge-triggered). The per-connection request cap workaround in `async_handle_one` was removed in rondo after this landed.

When you encounter a *new* compiler quirk, add an entry to `docs/riven-issues.md` (W-series for workarounds in code, F-series for fixes committed upstream).

## Conventions specific to this codebase

- **`&String` for read-only string params, `&var T` for mutable refs.** Map iteration uses `let ks = m.keys(); ks.get(i)` because `for k in m.keys()` hits W5.
- **Header names are lowercased on insertion** in both `Request.parse` and `Response.with_header` — this matches Sinatra's case-insensitive lookup and is what HTTP/2 mandates on the wire. Don't break this invariant.
- **`with_cookie` stores under key `set-cookie:<name>`** so multiple cookies don't overwrite each other in the headers Map. `Response.to_http` strips that prefix back out on emission.
- **HEAD requests fall back to the matching GET route**, then the response body is stripped at the wire layer in `handle_one` so `Content-Length` still reflects the GET-shape byte count (RFC 9110 §9.3.2).
- **Static-file fallback rejects `..` segments** in `try_static_or_404` to prevent path traversal — preserve this if you touch the static path.
- **Empty `public_folder` disables static serving.** Don't replace the check with a `nil` sentinel.
