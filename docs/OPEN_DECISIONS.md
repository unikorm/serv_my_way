# mini-nginx: Open Decisions

> Each item: the question, the options, my recommendation, and when it must be settled.
> Reply with a number and "agree" or your choice; anything unanswered before a
> milestone starts will follow the recommendation.

| # | Decision | Recommendation | Needed by |
|---|---|---|---|
| 1 | Do the config parser before the process model? | Yes: M3 before M2 | before M2/M3 |
| 2 | Worker spawn model | Re-exec + fd inheritance (`ExtraFiles`); `SO_REUSEPORT` opt-in later | M2 |
| 3 | Concurrency inside a worker | Goroutine per connection now; epoll loop as M11 experiment | M1 |
| 4 | How workers receive config | Validated config bytes over the control socket | M2 |
| 5 | Config shape | nginx style: `listen` inside `server`, listeners derived | M3 |
| 6 | `Host` header sent upstream | Forward the client's `Host` unchanged (nginx default is `$proxy_host`) | M1 |
| 7 | Parser strictness | Strict: reject bare LF, bare CR, obs-fold, whitespace before colon | M1 |
| 8 | State shared across workers | Per-worker; default `worker_processes 1`; shared memory is a stretch | M6 |
| 9 | Rate-limit rejection status | 429 with `Retry-After` (nginx: 503), configurable | M6 |
| 10 | `SIGTERM` semantics | Graceful with a deadline (nginx: fast) | M2 |
| 11 | Privileges | Run as one unprivileged user with `CAP_NET_BIND_SERVICE`; no privilege drop | M2 |
| 12 | Module path and binary name | Module path needs your GitHub user (see below); binary `mini-nginx` | M1 |
| 13 | Test backend may use `net/http` | Yes | M1 |
| 14 | Regex locations in M3 | Exact and prefix first; `~` regex via `regexp` as M3 stretch | M3 |
| 15 | HTTP/1.0 clients | Support (close semantics, no chunked to them) | M1 |
| 16 | Retries with streamed bodies | Accept the limitation: retry only before body bytes are sent | M7b |
| 17 | Request and response trailers | Consume and drop; not forwarded | M1 |
| 18 | ACME / automatic certificates | Out of scope; certbot-provisioned files | never, unless you want an M12 |

## 1. Config parser before process model

**Question.** Keep M2 (master/worker) before M3 (config), or swap them?

**Options.** (a) Agreed order. (b) Swap: M1, M3, M2, M4.

**Recommendation.** (b). The master's purpose is to validate config and hand it to
workers; reload means "new config file". With (a), M2's reload has nothing real to
reload and its tests are artificial; with (b), M2 lands as a complete, testable
feature, and you can route several projects sooner. Nothing built in either order is
thrown away.

## 2. Worker spawn model

**Options.** (a) Master opens sockets, workers inherit them through `ExtraFiles`.
(b) Each worker binds with `SO_REUSEPORT`.

**Recommendation.** (a). It mirrors nginx's default, has one accept queue, and loses
no connections on reload. (b) resets queued connections when a worker's socket closes.
(b) becomes `listen ... reuseport` in M11 to measure the thundering-herd cost of (a).
Details in [ARCHITECTURE.md](ARCHITECTURE.md) section 3.

## 3. Concurrency inside a worker

**Options.** (a) Goroutine per connection over the Go netpoller. (b) Hand-written
epoll loop from the start.

**Recommendation.** (a) now, (b) as M11 for plaintext only. (b) would double M1's
effort, forces state-machine code before the HTTP semantics are understood, and does
not compose with `crypto/tls`. The buffer-based parser API keeps (b) reachable.

## 4. How workers receive their configuration

**Options.** (a) Path plus environment; worker re-reads the file. (b) The master sends
the validated bytes over the control socket; the worker parses them. (c) The master
sends a serialized parsed `Config` (`encoding/gob`).

**Recommendation.** (b). The worker uses exactly what the master validated, even if
the file changed in between. (c) drags compiled regexes and TLS keys through gob for
no gain. The same channel can later carry certificate PEM bytes.

## 5. Config file shape

**Options.** (a) nginx shape: `server { listen 80; listen 443 ssl; server_name ...;
location ... }`; listeners are derived by grouping `listen` directives. (b) Your
original hierarchy as literal blocks: `listener 0.0.0.0:80 { server ... { location ... } }`.

**Recommendation.** (a). Existing nginx knowledge and docs transfer directly, a server
can listen on several ports without duplication, and the derived `Listener` type still
exists in the domain model exactly as you described it.

## 6. `Host` header sent to the upstream

**Options.** (a) nginx default: `Host` becomes the upstream's name
(`proxy_set_header Host $proxy_host`). (b) Forward the client's `Host` unchanged
(Caddy and Traefik default; what nearly every nginx config sets by hand with
`proxy_set_header Host $host`).

**Recommendation.** (b), configurable with `proxy_set_header Host ...`. Your backends
generate links and cookies from `Host`; forwarding it is what you want in practice.
This is a deliberate deviation from nginx; the doc notes it.

## 7. Parser strictness

**Options.** (a) Strict per RFC 9112 "MUST reject" rules, and additionally reject bare
LF line endings. (b) nginx-lenient: accept bare LF, tolerate some sloppiness.

**Recommendation.** (a) for M1. It keeps the parser small and every rejection is a
test case. Because we re-serialize requests, leniency would not create smuggling risk
toward backends, so we can relax individual rules later if a real client needs it.

## 8. State shared across workers

**Question.** nginx shares rate-limit zones, upstream health, and the cache index
across workers through shared memory. Do we?

**Options.** (a) Per-worker state; document that limits and health are per worker;
default `worker_processes 1` so it does not matter in your deployment. (b) A shared
memory region via `syscall.Mmap` on a file plus atomic operations and a fixed-size
hash table. (c) A separate "state" process over a Unix socket.

**Recommendation.** (a). (b) is a genuinely interesting stretch goal after M8, worth
maybe 15 hours, and the domain model's `Zone`/`Peer` types are designed so their
storage could move behind an interface. (c) adds latency to every request for
nothing.

## 9. Rate-limit rejection status

**Options.** (a) 503, nginx's default (`limit_req_status`). (b) 429 Too Many Requests
(RFC 6585 §4) with `Retry-After`.

**Recommendation.** (b), with `limit_req_status` to override. 429 is what clients and
dashboards expect today; 503 is confused with backend outages in logs.

## 10. `SIGTERM` semantics

**Options.** (a) nginx: `TERM` is a fast shutdown, `QUIT` is graceful. (b) `TERM` is
graceful with a deadline (`worker_shutdown_timeout`, default 10s), then `KILL`;
`INT` is fast.

**Recommendation.** (b). systemd and container runtimes send `TERM`; graceful should
be the default there. `QUIT` stays graceful without a deadline, matching nginx.

## 11. Privileges and port 80/443

**Options.** (a) nginx style: master runs as root, workers drop to `user www-data`
via `syscall.Credential`; master must then read certificates and pass them to
workers. (b) Everything runs as one unprivileged user; the systemd unit grants
`AmbientCapabilities=CAP_NET_BIND_SERVICE` (or `setcap cap_net_bind_service=+ep` on
the binary); certificate files are readable by that user.

**Recommendation.** (b) for your deployment. (a) is a good M2 stretch exercise (the
`Credential` field is one line), but the certificate-passing consequence is real work
for little safety gain on a single-user box.

## 12. Module path and binary name

**Question.** `go.mod` needs a module path. The repository is `serv_my_way`; the
project name we use everywhere is `mini-nginx`.

**Options.** (a) `module github.com/<your-github-user>/serv_my_way` (matches the repo,
lets you `go install` later). (b) `module mininginx` (short, no host, fine for an
app that is never imported).

**Recommendation.** (a) if you tell me the GitHub user; otherwise (b). The binary and
the `cmd/` directory are `mini-nginx` either way.

## 13. Test backend using `net/http`

**Question.** The "no `net/http`" rule targets the proxy path. May the test backend
in `test/backend` and the integration harness use `net/http`?

**Recommendation.** Yes for the backend (it plays your real applications). No for the
test *client* side: tests send hand-written bytes over `net.Dial`, so they exercise
the exact bytes we care about.

## 14. Regex locations

**Options.** (a) M3 ships `=`, prefix, `^~` only; `~` and `~*` are an M3 stretch using
`regexp`. (b) All four in M3.

**Recommendation.** (a). `regexp` is RE2: no backreferences or lookaround, but also
no catastrophic backtracking, which is a security win worth a sentence in the docs.

## 15. HTTP/1.0 clients

**Recommendation.** Support them: default `Connection: close`, never send chunked to
them, honour `Connection: keep-alive`. It is a dozen lines and some monitoring tools
still speak 1.0.

## 16. Retries with streamed request bodies

**Question.** Streaming the body means a request cannot be replayed to another
backend once body bytes have gone out. nginx buffers by default and can retry.

**Recommendation.** Accept it: retry on connect failure or on failure before the
first body byte is sent, only for idempotent methods, exactly as nginx behaves with
`proxy_request_buffering off`. A `proxy_request_buffering on` mode with a small
memory buffer (`client_body_buffer_size`) could be a later addition if you want
retries for small POSTs.

## 17. Trailers

**Recommendation.** Consume chunked trailers and drop them, both directions. Only
gRPC needs them and gRPC needs HTTP/2, which is out of scope. `TE: trailers` is
therefore stripped too.

## 18. ACME

**Recommendation.** Out of scope. Use certbot (or `lego`) to write PEM files, and
`mini-nginx -s reload` from a renewal hook. An ACME client in pure stdlib is feasible
(`crypto/ecdsa`, `encoding/json`, raw TCP for HTTP-01) but it is its own multi-week
milestone; we can revisit as M12 once the proxy is in production.
