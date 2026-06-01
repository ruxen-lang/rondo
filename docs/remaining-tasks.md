# Rondo Remaining Tasks

Audited against `src/**` (not just carried forward). Tracks what is
actually implemented and what is genuinely outstanding, including
protocol- and runtime-level dependencies that affect Rondo's behaviour.

Rondo ships Sinatra-shaped apps for static sites, JSON APIs, and
form-driven (urlencoded) endpoints today. What's left is middleware
polish, a few request/response surface gaps, protocol support, and
Ruxen runtime/reactor work.

## Implemented (verified in source)

- Routing: `get/post/put/delete/patch`; `:name` path params (single +
  multi-segment); pre-split literal patterns (`route.rx`).
- Dispatch pipeline (`rondo.rx`): match → merge params → CORS preflight
  short-circuit → before-hooks (Some short-circuits) → handler /
  HEAD-fallback / static / 404 → after-hooks → error-handler
  substitution → CORS headers.
- HEAD: falls back to the matching GET route; body stripped at the wire
  layer so Content-Length still reflects the GET shape.
- Request (`request.rx`): `parse` (method/path/query/headers/cookies/
  body), `param`/`header`/`cookie` accessors, `form` (urlencoded only).
  Query and cookie names/values are URL-decoded (`%xx` + `+`).
- Response (`response.rx`): `text`/`json`/`html`, `not_found`/
  `internal_error`, `with_status`/`with_header`/`with_body`,
  `with_cookie`/`with_secure_cookie`/`with_cookie_attrs`, `redirect`/
  `redirect_with_status`, `to_http` (header lowercasing, `Set-Cookie`,
  Content-Length, reason phrases).
- Hooks: `before` (Option short-circuit), `after` (rewrite), `on_error`
  (per-status substitution, last-wins).
- CORS middleware: `cors`/`cors_origin`/`cors_methods`/`cors_headers`/
  `cors_max_age`/`cors_credentials`/`cors_policy`; OPTIONS preflight →
  204; `Vary: Origin` when origin ≠ `*`.
- Static files: `set_public_folder`, MIME-by-extension, `..` traversal
  rejection, empty-folder disables serving.
- Server: `host`/`port`/`workers`/`max_connections` config, `bind_addr`,
  bounded connection cap; `listen` overloads + `run`.
- Async runtime (Rondo-controlled, `async_server.rx`): thread-per-core
  SO_REUSEPORT workers, HTTP/1.1 keep-alive request loop, 60 s
  per-connection idle timeout (`read_with_timeout`), `Task.spawn_raw`
  per-connection concurrency, bounded active-connection cap.
- Tests: a real `ruxen test` suite — `tests/{dispatch,request,response,
  route,router,helpers,cors,config}_test.rx` (61 cases). `Rondo.dispatch`
  remains the pure, socket-free seam. (The async server loop itself is
  not unit-testable in `ruxen test`'s fork model — see CLAUDE.md.)

## Remaining — framework surface

- [ ] Request ID middleware
- [ ] Structured logging middleware
- [ ] Multipart `multipart/form-data` parsing + file uploads —
      `req.form` handles **only** `application/x-www-form-urlencoded`
      today.
- [ ] Regex / constraint / wildcard route patterns — routing is literal
      segments + `:name` only. Now feasible: Ruxen ships `std.regex`
      (PCRE2: `Regex.new`, `is_match`, `find`, `captures`). Smallest
      useful increment: per-param constraints (e.g. `:id` ⇒ `\d+`),
      keeping the fast literal path for plain routes.
- [ ] Explicit `head` / `options` route registration — HEAD is served
      via GET fallback and OPTIONS only via the CORS preflight; there is
      no user-facing `app.head(...)` / `app.options(...)`.
- [ ] Response streaming / chunked transfer-encoding — responses are
      fully buffered (`to_http` builds one String).
- [ ] Request size limits + `413`/`431` — the async reader caps the head
      at 16 KB and total at 1 MB and then silently drops; no configurable
      limit or status response.
- [ ] WebSocket support — protocol-level, dedicated effort.
- [ ] TLS / HTTPS support — protocol-level.
- [ ] HTTP/2 support — protocol-level.

## Ruxen runtime / reactor (audited against the Ruxen source)

Verified against `library/std/future/runtime/{reactor,scheduler}.c`,
`library/std/async_net/runtime/async_net.c`, and
`library/std/sync/runtime/thread.c` (Ruxen checkout). Tracked here
because incomplete reactor scheduling surfaces as Rondo latency/
throughput regressions under concurrent keep-alive load.

Done:

- [x] Per-task waker routing / selective ready queue — `cx.waker().wake()`
      marks only that task's queue entry ready; each queued task owns a
      stable Context/Waker (`scheduler.c`).
- [x] Edge-triggered retry-on-EAGAIN for async TCP I/O — the reactor parks
      on EAGAIN and re-arms; macOS uses `EV_CLEAR` (`reactor.c`,
      `async_net.c` status codes 0=progress/1=EAGAIN/2=done/3=error).
- [x] Drop-order guarantee: deregister reactor handle before closing fd —
      `ruxen_async_tcp_stream_drop` unregisters, then closes (`async_net.c`,
      "Deregister BEFORE close").
- [x] Worker-thread stack-size default + config — 512 KB default,
      overridable via `RUXEN_THREAD_STACK_KB` (`sync/runtime/thread.c`).
      Note: that's an env knob, not a Rondo builder-surface option.

Still open:

- [ ] Real task waker ABI wired through fd + timer registrations — the
      Waker ABI exists, but async-net/timer futures still use reactor
      readiness directly rather than storing per-fd/timer wakers
      (`scheduler.c` note). Coupled with the next item.
- [ ] Remove compatibility wake-all paths from socket/timer readiness —
      OS readiness still "falls back to marking all queued tasks ready"
      (`scheduler.c`). Until fd/timer slots carry wakers, every ready
      event wakes the whole queue.
- [ ] Configurable per-worker max in-flight requests — not in Ruxen;
      Rondo exposes only a global `max_connections` cap, not a per-worker
      in-flight limit.
- [ ] Keep-alive-aware metrics: active / idle / requests-per-connection —
      no counters in the runtime or in Rondo.

## Notes

- WebSockets, TLS/HTTPS, and HTTP/2 are protocol-level efforts — plan
  each as dedicated work, not incidental server edits.
- Persistent fd registration (edge-triggered) and `Task.spawn_raw`
  per-connection concurrency landed in Ruxen and are in use; the waker
  items above are the remaining runtime correctness work.
