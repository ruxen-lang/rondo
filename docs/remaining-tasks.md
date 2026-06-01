# Rondo Remaining Tasks

This file tracks the remaining Rondo work at the framework boundary.
It intentionally includes protocol-level and runtime-level dependencies
so they do not disappear from planning between Rondo and Ruxen sessions.

Rondo is complete enough to ship Sinatra-shaped apps for static sites,
JSON APIs, and form-driven endpoints. The remaining items are either
framework middleware polish, protocol support, or Ruxen runtime work.

## Framework Surface

- [x] CORS middleware
- [ ] Request ID middleware
- [ ] Structured logging middleware
- [ ] WebSocket support
- [ ] TLS/HTTPS support
- [ ] HTTP/2 support

## Runtime And Async Dependencies

These come from `docs/research/async-threading-best-practices.md` and
are tracked here because they directly affect Rondo server behaviour.

- [x] HTTP/1.1 keep-alive request loop
- [x] Per-connection idle timeout
- [x] Persistent fd registration for async TCP streams
- [x] `task::spawn` / `Task.spawn_raw` based per-connection concurrency
- [x] Bounded active connection cap
- [ ] Real task waker ABI wired through fd and timer registrations
- [ ] Per-task waker routing / selective ready queue
- [ ] Remove compatibility wake-all paths from socket/timer readiness
- [ ] Edge-triggered retry-on-EAGAIN loops for all async TCP I/O paths
- [ ] Drop order guarantee: deregister reactor handle before closing fd
- [ ] Configurable per-worker max in-flight requests
- [ ] Keepalive-aware metrics: active, idle, and requests-per-connection
- [ ] Stack size default and builder/config surface for worker threads

## Notes

- WebSockets, TLS/HTTPS, and HTTP/2 are protocol-level efforts and should
  be planned as dedicated work, not incidental server edits.
- The waker/ready-queue items belong primarily in Ruxen, but Rondo should
  keep tracking them because incomplete runtime scheduling shows up as
  latency and throughput regressions under concurrent keep-alive load.
