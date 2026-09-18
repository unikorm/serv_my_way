# mini-nginx: Roadmap

> Status: Phase 1 design draft. Milestones are in the order we agreed. Section 1
> proposes two adjustments; the rest assumes the agreed order unless you accept them.
> Effort assumes 5 hours per week, learning Go along the way, writing tests, and
> reading the referenced RFC sections. An experienced Go developer would need roughly
> 40% of these numbers.

## 1. Suggested adjustments (your call, see OPEN_DECISIONS #1)

1. **Do the config parser (M3) before the process model (M2).** The master's main job
   is "parse and validate config, then hand it to workers", and "reload" means "read a
   new config file". With M2 first, the master has nothing to parse and reload only
   re-reads command-line flags. With M3 first, M2 becomes a real reload from day one
   and you can put several projects behind the proxy sooner. No code is wasted either
   way; the swap only changes what M2's tests look like.
2. **Split the larger milestones into a/b halves** so each half has its own
   definition of done: M5a TLS + SNI, M5b upstream pooling + TLS to backends;
   M7a balancing + passive health, M7b active health + retries; M8a memory cache,
   M8b disk cache. The estimates below are already per half.

Everything else stays as agreed. Milestone 4 (timeouts) intentionally comes after the
process model because drain behaviour depends on timeouts existing.

## Milestone 1: Raw TCP, hand-written HTTP/1.1, single-backend proxy

**Goal.** `mini-nginx -upstream 127.0.0.1:3000` listens on a port, parses HTTP/1.1
requests from raw bytes, forwards them to one backend, streams the response back, and
keeps client connections alive. No config file, no TLS, no timeouts yet.

**What you will learn.** How HTTP/1.1 looks on the wire; request framing and why it is
the root of request smuggling; hop-by-hop vs end-to-end headers; chunked encoding;
Go's `net.Conn`, slices as buffer views, `sync.Pool`, goroutines, table-driven tests
and fuzzing.

**Key concepts and reading.**

- RFC 9112: §2.1 message format, §2.2 message parsing (bare CR/LF), §3 request line
  and §3.2 request target forms, §5 field syntax (§5.1 whitespace before colon,
  §5.2 obs-fold), §6 message body (§6.1 Transfer-Encoding, §6.2 Content-Length,
  §6.3 message body length: the smuggling rules), §7.1 chunked coding, §9.3
  persistence, §9.3.2 pipelining, §9.6 tear-down, §11.1 request smuggling,
  §11.2 response splitting.
- RFC 9110: §5.3 field order, §5.5 field values, §7.6.1 `Connection`, §7.6.3 `Via`,
  §7.7 message transformations, §8.6 Content-Length, §9.2.2 idempotent methods,
  §10.1.1 `Expect`, §15.2 informational responses, §15.6.3 502, §15.6.5 504.
- RFC 7239 `Forwarded` (we implement the de-facto `X-Forwarded-*` family, but read it).
- nginx: `ngx_http_parse.c` (`ngx_http_parse_request_line`,
  `ngx_http_parse_header_line`), `ngx_http_request.c`
  (`ngx_http_process_request_headers`, `ngx_http_set_keepalive`),
  `ngx_http_proxy_module.c` (`ngx_http_proxy_create_request`,
  `ngx_http_proxy_process_header`, `ngx_http_proxy_chunked_filter`),
  `ngx_http_chunked_filter_module.c`. Directives: `proxy_pass`, `proxy_set_header`,
  `proxy_http_version`, `proxy_request_buffering`, `proxy_buffering`,
  `keepalive_requests`, `tcp_nodelay`.

**Steps (one concept each; every step ends runnable).**

1. Accept loop: `net.Listen`, `Accept`, one goroutine per connection, log the raw bytes
   received, answer a fixed `200 OK`. `curl -v` works.
2. `buf.Buffer` and "find end of head": read into a fixed buffer until `\r\n\r\n` or full.
3. Request-line parser with byte fixtures: method token, target forms, version.
4. Field-line parser and `Headers`: token names, OWS trimming, rejects (whitespace
   before colon, obs-fold, control chars), count and size caps.
5. Framing decision (`BodyFraming`) with the full §6.3 table as a test table.
6. `LengthReader`; `ChunkedReader` (sizes, extensions, trailers); `ChunkedWriter`;
   the "split at every byte" incremental test.
7. Request serializer; dial the backend; write head; stream body; read and parse
   the response head (status line parser).
8. Response framing; stream body to client; re-chunk close-delimited bodies for
   HTTP/1.1 clients; `HEAD`/`204`/`304` rules.
9. Hop-by-hop stripping, `Connection:`-listed headers, `X-Forwarded-For/Proto/Host`,
   `Host` handling.
10. Client keep-alive loop with leftover bytes (pipelining), HTTP/1.0 semantics,
    local error responses with `Connection: close`.
11. Integration test suite in the VM against a Go test backend; smuggling fixtures over
    `nc`.

Stretch: forward `1xx` responses and `Expect: 100-continue`.

**Simplifications made in M1 and when they go away.** One new TCP connection per
request to the backend with `Connection: close` (pooling in M5b). Body upload and
response read are sequential (concurrent in M7b). No timeouts (M4). Trailers dropped
(not planned). Backend from a flag (M3).

**Definition of done.** With a Go backend on `:3000` that echoes received headers and
supports `/slow` (1 byte per second for 10 s) and `/nolength` (close-delimited body):

```sh
# basic proxying, XFF present in the echoed headers
curl -sv http://localhost:8080/echo | grep -i x-forwarded-for

# streaming: bytes arrive progressively, not after 10 s
curl -sN http://localhost:8080/slow | ts '%.s'          # or: | while read -n1 c; do date +%T.%N; done

# large bodies are streamed, both framings, and the backend sees a sane request
head -c 50000000 /dev/urandom > /tmp/big
curl -sv --data-binary @/tmp/big http://localhost:8080/echo -o /dev/null   # Content-Length
curl -sv -H 'Transfer-Encoding: chunked' --data-binary @/tmp/big http://localhost:8080/echo -o /dev/null

# close-delimited upstream body reaches an HTTP/1.1 client as chunked
curl -sv http://localhost:8080/nolength 2>&1 | grep -i 'transfer-encoding: chunked'

# client keep-alive: two requests on one connection ("Re-using existing connection")
curl -sv http://localhost:8080/a http://localhost:8080/b -o /dev/null 2>&1 | grep -i re-using

# HTTP/1.0 client gets Connection: close
curl -sv --http1.0 http://localhost:8080/ -o /dev/null 2>&1 | grep -i 'connection: close'

# smuggling and syntax fixtures, each must answer 400 and close
printf 'GET / HTTP/1.1\r\nHost: x\r\nContent-Length: 5\r\nTransfer-Encoding: chunked\r\n\r\nhello' | nc -q1 localhost 8080
printf 'GET / HTTP/1.1\r\nHost: x\r\nContent-Length: 5\r\nContent-Length: 6\r\n\r\n' | nc -q1 localhost 8080
printf 'GET / HTTP/1.1\r\n\r\n' | nc -q1 localhost 8080                       # missing Host
printf 'GET / HTTP/1.1\r\nHost: x\r\nFoo : bar\r\n\r\n' | nc -q1 localhost 8080  # space before colon
printf 'GET / HTTP/1.1\r\nHost: x\r\nTransfer-Encoding: gzip, chunked\r\n\r\n' | nc -q1 localhost 8080

# hop-by-hop: X-Secret listed in Connection must not reach the backend
curl -s -H 'Connection: X-Secret' -H 'X-Secret: 1' http://localhost:8080/echo | grep -ci x-secret   # 0

# pipelining: two requests in one write, two responses in order
printf 'GET /a HTTP/1.1\r\nHost: x\r\n\r\nGET /b HTTP/1.1\r\nHost: x\r\n\r\n' | nc -q1 localhost 8080

# backend down -> 502
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:8080/    # 502

# unit tests, race detector, fuzzing for a minute
go test -race ./...
go test -fuzz=FuzzParseRequestHead -fuzztime=60s ./internal/http1
```

**Effort.** 30 hours, about 6 weeks.

## Milestone 2: Master/worker, signals, graceful shutdown and reload

**Goal.** `mini-nginx` starts a master that opens the listeners, spawns `N` workers
by re-executing itself with inherited sockets, restarts crashed workers, drains on
`SIGQUIT`/`SIGTERM`, and reloads on `SIGHUP` with zero failed requests.

**What you will learn.** Why Go cannot fork; fd inheritance across `exec`; Unix
signals and `os/signal`; `socketpair`; process supervision; connection draining;
`Pdeathsig`; pid files and `-s reload`.

**Key concepts and reading.** `fork(2)` vs `execve(2)`; `signal(7)`; `unix(7)` and
`SCM_RIGHTS` (optional); `prctl(2)` `PR_SET_PDEATHSIG`; systemd socket activation
(`sd_listen_fds(3)`) as the same idea; Go `os/exec.Cmd.ExtraFiles`,
`net.FileListener`, `syscall.SysProcAttr`. nginx: `ngx_process_cycle.c`
(`ngx_master_process_cycle`, `ngx_start_worker_processes`, `ngx_reap_children`),
`ngx_channel.c`, `ngx_process.c` (`ngx_signal_handler`); directives
`worker_processes`, `worker_shutdown_timeout`, `pid`, `daemon`; the "Controlling
nginx" chapter of the docs.

**Steps.**

1. `worker` subcommand; master opens the listener, spawns one worker with
   `ExtraFiles`, worker rebuilds the listener from fd 4 and serves.
2. Control socket: master sends the config frame (M1: the flag values; M3: the file
   bytes), worker replies `READY`, worker exits on EOF.
3. Signals in the master: `SIGCHLD` reaping and respawn with backoff; `SIGQUIT` and
   `SIGTERM` propagate; pid file; `-s quit|stop|reload|reopen`.
4. Worker drain: stop accepting, close idle keep-alives, finish in-flight requests
   with `Connection: close`, exit; master enforces `worker_shutdown_timeout`.
5. `SIGHUP`: new generation, wait for `READY`, drain the old one, roll back on failure.
6. `worker_processes N`; observe several workers accepting from one socket.

Stretch: `Pdeathsig`; `user` directive via `syscall.Credential`; a `SO_REUSEPORT`
listener variant behind a flag (measured properly in M11).

**Definition of done.**

```sh
# zero failed requests across five reloads under load (M3 config or M1 flags)
wrk -t2 -c100 -d30s http://localhost:8080/ &
for i in 1 2 3 4 5; do sleep 4; kill -HUP "$(cat /run/mini-nginx.pid)"; done; wait
# wrk output: "Socket errors: connect 0, read 0, write 0, timeout 0", "Non-2xx: 0"

# new worker pids after reload
ps -o pid,ppid,etime,args -C mini-nginx

# graceful shutdown finishes an in-flight slow request
curl -sN http://localhost:8080/slow & sleep 1; kill -QUIT "$(cat /run/mini-nginx.pid)"; wait  # full body

# crashed worker is respawned within a second
kill -9 <worker pid>; sleep 1; ps -C mini-nginx

# master killed -> workers exit on their own
kill -9 <master pid>; sleep 1; pgrep mini-nginx   # nothing

# invalid config on reload keeps the old generation running
```

**Effort.** 20 hours, about 4 weeks.

## Milestone 3: Config file parser, host- and path-based routing

**Goal.** An nginx-style config with `upstream`, `server`, `listen`, `server_name`,
`location`, `proxy_pass`, `return`, and global directives, parsed by a hand-written
lexer and parser into a typed `Config`, with `mini-nginx -t` reporting errors with
line and column.

**What you will learn.** Lexing and recursive-descent parsing; separating syntax
(directive tree) from semantics (schema and validation); nginx's `listen` grouping,
`server_name` matching order, `location` precedence, and `proxy_pass` URI rewriting.

**Key concepts and reading.** nginx docs: "Beginner's guide", "How nginx processes a
request", "Server names", `ngx_http_core_module` (`listen`, `server_name`, `location`,
`default_server`), `ngx_http_proxy_module` (`proxy_pass` URI semantics),
`ngx_http_upstream_module` (`upstream`, `server`). nginx source: `ngx_conf_file.c`
(`ngx_conf_parse`, `ngx_conf_read_token`), `ngx_http.c` (`ngx_http_init_locations`,
`ngx_http_add_addresses`). RFC 3986 §3.3 paths.

**Steps.**

1. Lexer: words, quoted strings, `;`, `{`, `}`, `#` comments, positions.
2. Parser to `[]Directive`; tests with fixtures including error cases.
3. Schema: walk the tree into `Config`; unknown directive, wrong arity, duplicate,
   and context errors with positions; `-t`.
4. Derive `Listener`s by grouping `listen` across servers; default server rule.
5. `Router`: exact, wildcard, default host matching; `location` precedence
   (`=`, `^~`, longest prefix).
6. `proxy_pass` with and without a URI part; `return`.
7. Wire the master to send the file bytes and the worker to parse them.

Stretch: regex locations with `regexp` (RE2 semantics; document the PCRE differences);
`include`; size and duration literals (`10m`, `30s`) done properly.

**Definition of done.**

```sh
mini-nginx -t -c bad.conf          # "bad.conf:12:5: unknown directive "proxy_pas""; exit 1
curl -s -H 'Host: a.test' http://localhost:8080/       # routed to upstream A
curl -s -H 'Host: b.test' http://localhost:8080/       # routed to upstream B
curl -s -H 'Host: nope'   http://localhost:8080/       # default server
curl -s -H 'Host: a.test' http://localhost:8080/api/x  # location /api/ -> upstream C, path preserved
curl -s -H 'Host: a.test' http://localhost:8080/old/x  # location /old/ with proxy_pass http://c/new/ -> backend sees /new/x
curl -s -o /dev/null -w '%{http_code}' -H 'Host: a.test' http://localhost:8080/health  # return 200
```

**Effort.** 15 hours, about 3 weeks.

## Milestone 4: Timeouts and limits

**Goal.** Every blocking point has a deadline and every input has a cap, configurable
globally, per server, and per location.

**What you will learn.** `SetReadDeadline`/`SetWriteDeadline` semantics (absolute
deadlines, re-armed per operation); Slowloris and slow-body attacks; why a proxy needs
different timeouts on each side; per-worker connection accounting.

**Key concepts and reading.** nginx `client_header_timeout`, `client_body_timeout`,
`keepalive_timeout`, `send_timeout`, `client_max_body_size`,
`large_client_header_buffers`, `worker_connections`, `proxy_connect_timeout`,
`proxy_read_timeout`, `proxy_send_timeout`, `limit_conn`. RFC 9110 §15.5 (408, 413),
RFC 6585 §5 (431). `tcp(7)` `TCP_DEFER_ACCEPT` (nginx `listen ... deferred`).

**Steps.**

1. `Limits` type, merge rules (global to server to location), config directives.
2. Header timeout and idle timeout on the client read; 408 vs silent close.
3. Body timeout (per read) and `client_max_body_size` (413, including chunked).
4. Send timeout on client writes; upstream connect, send, and read timeouts (504).
5. `ConnGate` for `worker_connections`; 503 when exhausted; `limit_conn` per IP.

**Definition of done.**

```sh
# Slowloris: headers trickle -> 408 after client_header_timeout (set to 3s)
(printf 'GET / HTTP/1.1\r\nHost: x\r\n'; sleep 5; printf 'X: y\r\n\r\n') | nc -q1 localhost 8080

# idle keep-alive closed after keepalive_timeout (set to 2s)
(printf 'GET / HTTP/1.1\r\nHost: x\r\n\r\n'; sleep 4; printf 'GET / HTTP/1.1\r\nHost: x\r\n\r\n') | nc -q1 localhost 8080   # one response, then closed

# header too large -> 431
python3 -c "print('GET / HTTP/1.1\r\nHost: x\r\nX: ' + 'a'*20000 + '\r\n\r\n', end='')" | nc -q1 localhost 8080

# body too large -> 413 (client_max_body_size 1m)
curl -s -o /dev/null -w '%{http_code}' --data-binary @/tmp/big http://localhost:8080/

# slow backend -> 504 after proxy_read_timeout
curl -s -o /dev/null -w '%{http_code}' http://localhost:8080/hang

# connection cap -> 503 for the 11th connection with worker_connections 10
```

**Effort.** 10 hours, about 2 weeks.

## Milestone 5a: TLS termination with hand-parsed SNI

**Goal.** `listen 443 ssl` with per-`server` certificates; the certificate is selected
by SNI extracted by our own ClientHello parser; the connection is then handed to
`crypto/tls` untouched.

**What you will learn.** The TLS record layer and handshake framing; the ClientHello
layout; SNI (RFC 6066) and ALPN (RFC 7301); how TLS 1.3 hides its version; what
`crypto/tls` needs from a `net.Conn`; wildcard certificate matching.

**Key concepts and reading.** RFC 8446 §4.1.2 ClientHello, §4.2 extensions,
§4.2.1 supported_versions, §5.1 record layer (fragmentation across records);
RFC 6066 §3 server_name; RFC 7301 §3.1 ALPN; RFC 5246 §7.4.1.2 for the TLS 1.2 view.
`openssl s_client -msg` to capture real ClientHello bytes. nginx `ngx_http_ssl_module`
(`ssl_certificate`, `ssl_protocols`, `ssl_alpn`), `ngx_http_request.c`
(`ngx_http_ssl_servername`).

**Steps.**

1. Capture hex fixtures of ClientHellos (curl, Chrome, `openssl s_client`, with and
   without SNI, TLS 1.2 and 1.3, one spanning two records).
2. `ParseClientHello`: record header, handshake header, vectors, extensions; tests.
3. `CertStore` from `ssl_certificate`/`ssl_certificate_key` per server; wildcards.
4. `prefixConn`; `tls.Server` with the chosen certificate, `NextProtos: ["http/1.1"]`,
   `MinVersion: TLS 1.2`; handshake deadline.
5. `X-Forwarded-Proto: https`, `$ssl_protocol`, HTTP-to-HTTPS `return 301`.
6. Cross-check against `tls.Config.GetCertificate`'s `ServerName` in tests.

**Definition of done.**

```sh
openssl s_client -connect localhost:8443 -servername a.test </dev/null 2>/dev/null | openssl x509 -noout -subject   # CN=a.test
openssl s_client -connect localhost:8443 -servername b.test </dev/null 2>/dev/null | openssl x509 -noout -subject   # CN=b.test
openssl s_client -connect localhost:8443 </dev/null 2>/dev/null | openssl x509 -noout -subject                     # default cert
openssl s_client -connect localhost:8443 -servername a.test -tls1_2 </dev/null   # works
openssl s_client -connect localhost:8443 -servername a.test -tls1_1 </dev/null   # refused
curl -sv --resolve a.test:8443:127.0.0.1 --cacert ca.pem https://a.test:8443/echo | grep -i 'x-forwarded-proto: https'
printf 'garbage' | nc -q1 localhost 8443     # closed quickly, no crash, no goroutine leak
```

**Effort.** 12 hours, about 2.5 weeks.

## Milestone 5b: Upstream connection pooling, keep-alive and TLS to backends

**Goal.** Reuse backend connections across requests (`keepalive N` in `upstream`),
detect stale pooled connections safely, and support `proxy_pass https://` with
certificate verification.

**What you will learn.** Connection pool design (idle list, max idle, idle timeout);
the "stale connection" race and why only some requests may be retried; `tls.Client`,
`RootCAs`, `ServerName`.

**Key concepts and reading.** RFC 9112 §9.3.1 retrying requests, §9.6 tear-down; RFC 9110 §9.2.2 idempotent methods. nginx `ngx_http_upstream_module`
`keepalive`, `keepalive_requests`, `keepalive_timeout`; `proxy_ssl_verify`,
`proxy_ssl_trusted_certificate`, `proxy_ssl_server_name`, `proxy_ssl_name`;
`ngx_http_upstream_keepalive_module.c`.

**Steps.**

1. `upstream.Pool.Get/Put`, idle list per peer, bounded by `keepalive`; idle timeout
   sweeper; `Connection: keep-alive` upstream; keep reading the response fully before `Put`.
2. Stale detection: a pooled connection that fails on first write or yields EOF before
   any response byte is retried once on a fresh connection, only for idempotent methods
   with no body bytes sent yet.
3. `tls.Client` dialer with verification options.

**Definition of done.**

```sh
# one backend connection serves many requests
wrk -t2 -c50 -d10s http://localhost:8080/ ; ss -tn state established '( dport = :3000 )' | wc -l   # about keepalive N, not thousands
# backend restarts under load -> at most a handful of 502s, no hangs
# https backend with self-signed cert: proxy_ssl_verify on fails, off works, trusted CA works
```

**Effort.** 12 hours, about 2.5 weeks. **Deployment checkpoint:** after M5 the proxy
can sit in front of your projects with real certificates (obtained by certbot; ACME
is out of scope) behind a systemd unit with `AmbientCapabilities=CAP_NET_BIND_SERVICE`
and `ExecReload=/bin/kill -HUP $MAINPID`.

## Milestone 6: Rate limiting

**Goal.** `limit_req_zone $binary_remote_addr zone=z:10m rate=10r/s;` and
`limit_req zone=z burst=20 nodelay;` semantics, per worker.

**What you will learn.** Leaky bucket vs token bucket; nginx's integer millisecond
arithmetic; `burst`, `delay`, `nodelay`; keying by client IP; bounded tables and
expiry; why per-worker limits differ from nginx's shared-memory zones.

**Key concepts and reading.** nginx blog "Rate Limiting with NGINX"; `ngx_http_limit_req_module.c`
(`ngx_http_limit_req_lookup`, the `excess` computation); RFC 6585 §4 (429);
RFC 9110 §10.2.3 `Retry-After`.

**Steps.**

1. `Zone` and `Bucket` with nginx's arithmetic
   (`excess = max(0, excess - rate*elapsed_ms/1000) + 1000; reject if excess > burst*1000`).
2. `limit_req` on locations; `Delay(d)` via `time.Sleep` in the connection goroutine
   (that is what an nginx timer does); `nodelay`; `delay=N`.
3. Bounded map with LRU expiry; `limit_req_status`, `Retry-After`; `limit_req_log_level`.

**Definition of done.**

```sh
hey -n 100 -c 10 -q 0 http://localhost:8080/   # rate=10r/s burst=5 nodelay: 5 immediate 200s plus ~1 per 100 ms, rest 429
hey -n 100 -c 10 -q 0 http://localhost:8080/   # rate=10r/s burst=20 (no nodelay): ~20 x 200 spread over ~2 s, rest 429
for i in $(seq 1 30); do curl -s -o /dev/null -w '%{http_code}\n' http://localhost:8080/; done | sort | uniq -c
```

**Effort.** 8 hours, about 2 weeks.

## Milestone 7a: Load balancing and passive health checks

**Goal.** `upstream` with several `server` entries, `weight`, `max_fails`,
`fail_timeout`, `backup`; policies round-robin (smooth weighted), `least_conn`,
optional `ip_hash`.

**What you will learn.** nginx's smooth weighted round-robin algorithm and why naive
WRR bursts; least-connections bookkeeping; passive health state transitions.

**Key concepts and reading.** `ngx_http_upstream_round_robin.c`
(`ngx_http_upstream_get_peer`), `ngx_http_upstream_least_conn_module.c`,
`ngx_http_upstream_module` docs (`server` parameters, `least_conn`, `ip_hash`,
`backup`).

**Steps.**

1. `Balancer` interface and `RoundRobin` with smooth weighting; unit tests asserting
   the exact nginx sequence for weights `5,1,1` (`a a b a c a a`).
2. `Peer.active` and `LeastConn`.
3. Passive health: failures increment `fails`, `max_fails` within `fail_timeout` marks
   the peer down for `fail_timeout`; `backup` peers used only when all primaries are down.

**Definition of done.**

```sh
for i in $(seq 1 14); do curl -s http://localhost:8080/whoami; done   # weights 5,1,1 -> a a b a c a a a a b a c a a
docker stop backend-b; for i in $(seq 1 5); do curl -s -o /dev/null -w '%{http_code}\n' http://localhost:8080/; done  # first 502 (max_fails 1) then only a and c
# after fail_timeout, b is tried again
```

**Effort.** 10 hours, about 2 weeks.

## Milestone 7b: Active health checks and retries

**Goal.** A prober per upstream (`health_check interval=5s fails=3 passes=2 uri=/healthz`),
and `proxy_next_upstream error timeout http_502 http_503` with
`proxy_next_upstream_tries`, plus the concurrent body upload so early responses are
noticed.

**What you will learn.** Retry safety with streamed bodies; idempotency; goroutine
coordination for full-duplex proxying; open-source nginx has no active checks (Plus
only) so this is a place we go beyond it.

**Key concepts and reading.** RFC 9110 §9.2.2, RFC 9112 §9.3.1; nginx
`proxy_next_upstream`, `proxy_next_upstream_timeout`, `proxy_next_upstream_tries`,
`ngx_http_upstream.c` (`ngx_http_upstream_next`).

**Definition of done.**

```sh
# kill backend b's process (connection refused) -> no client sees an error; log shows "next upstream"
# backend returning 503 for POST is NOT retried (non-idempotent) unless configured
# health check flips a peer to down within fails*interval; back to up within passes*interval
# early 413 from backend while uploading 50 MB -> client gets 413 promptly
```

**Effort.** 10 hours, about 2 weeks.

## Milestone 8a: In-memory cache

**Goal.** `proxy_cache` for `GET`/`HEAD` with RFC 9111 semantics: keys, freshness from
`Cache-Control`/`Expires`/`proxy_cache_valid`, `no-store`/`private`/`Authorization`
rules, `Vary`, conditional `304` to clients, `X-Cache-Status`, LRU with a byte budget.

**What you will learn.** HTTP caching semantics; tee-while-streaming; validators;
why `Vary` is hard; nginx `proxy_cache_key` defaults.

**Key concepts and reading.** RFC 9111 §3 storing responses, §4.1 `Vary`, §4.2
freshness (§4.2.1 lifetime, §4.2.2 heuristics, §4.2.3 age), §4.3 validation,
§5.2.2 response directives; RFC 9110 §13 conditional requests. nginx
`proxy_cache`, `proxy_cache_key`, `proxy_cache_valid`, `proxy_cache_bypass`,
`proxy_no_cache`, `proxy_cache_min_uses`, `$upstream_cache_status`.

**Definition of done.**

```sh
curl -sI http://localhost:8080/static/a.css | grep X-Cache-Status   # MISS then HIT
curl -sI http://localhost:8080/api/private | grep X-Cache-Status    # BYPASS (Cache-Control: no-store)
sleep 11; curl -sI ... | grep X-Cache-Status                         # EXPIRED then HIT (max-age=10)
curl -sI -H 'If-None-Match: "abc"' http://localhost:8080/static/a.css   # 304 served from cache, backend not called
curl -sI -H 'Accept-Encoding: gzip' ... ; curl -sI ...               # two variants when backend sends Vary: Accept-Encoding
```

**Effort.** 12 hours, about 2.5 weeks.

## Milestone 8b: On-disk cache

**Goal.** `proxy_cache_path /var/cache/mini-nginx levels=1:2 max_size=1g inactive=60m`
with a header block plus body per file, an index rebuilt at startup, size-based
eviction, and temp-file-then-rename writes.

**What you will learn.** Durable cache layout, atomic file replacement, directory
sharding, background eviction; nginx's cache loader and manager processes and why we
run them as goroutines.

**Reading.** `ngx_http_file_cache.c`; `rename(2)` atomicity; `fsync` trade-offs.

**Definition of done.** Cache survives `kill -HUP` and a full restart; `du` stays
under `max_size`; entries unused for `inactive` disappear; `ls` shows the
`levels=1:2` layout.

**Effort.** 12 hours, about 2.5 weeks.

## Milestone 9: Access logs and metrics

**Goal.** `log_format` with nginx variables, `access_log` per server/location,
buffered `O_APPEND` writes shared by workers, reopen on `SIGUSR1`; a `stub_status`
style endpoint plus a Prometheus text exposition.

**What you will learn.** Variable interpolation; append-only file semantics across
processes; log rotation protocols; what "reading/writing/waiting" mean.

**Reading.** nginx `ngx_http_log_module` (`log_format`, `access_log`, `combined`),
`ngx_http_stub_status_module`, `ngx_http_variables.c`; `logrotate` `postrotate`.
Variables to support: `$remote_addr $time_local $request $status $body_bytes_sent
$http_referer $http_user_agent $request_time $upstream_addr $upstream_response_time
$upstream_cache_status $host $server_name $ssl_protocol $request_length`.

**Definition of done.**

```sh
tail -1 /var/log/mini-nginx/access.log   # matches nginx "combined" byte for byte
mv access.log access.log.1; kill -USR1 $(cat /run/mini-nginx.pid); curl ...; tail -1 access.log   # new file receives the line
curl -s http://localhost:8080/status      # "Active connections: 3 ... Reading: 0 Writing: 1 Waiting: 2"
curl -s http://localhost:8080/metrics | grep mininginx_requests_total
```

**Effort.** 10 hours, about 2 weeks.

## Milestone 10: WebSocket proxying

**Goal.** Detect `Upgrade: websocket`, forward the upgrade to the upstream, and on
`101` switch the connection to a bidirectional byte tunnel with half-close and idle
timeout.

**What you will learn.** RFC 6455 opening handshake; RFC 9110 §7.8 `Upgrade` as the
one hop-by-hop header a proxy must forward on purpose; full-duplex copying with two
goroutines; `CloseWrite`; leftover bytes on both sides after the head; `splice`.

**Reading.** RFC 6455 §1.3, §4.1, §4.2; RFC 9110 §7.8, §15.2.2 (101); nginx
"WebSocket proxying" page (`proxy_set_header Upgrade $http_upgrade; Connection "upgrade";`),
`ngx_http_upstream.c` upgraded connection handling.

**Definition of done.**

```sh
# node ws echo server on :3001; websocat through the proxy
echo hello | websocat ws://localhost:8080/ws          # hello
# 1000 messages both ways, close from client and from server, idle timeout closes both sides
```

**Effort.** 6 hours, about 1.5 weeks.

## Milestone 11 (optional): Hand-written epoll loop and benchmarks

**Goal.** A `worker_mode epoll` variant for plaintext listeners: `epoll_create1`,
non-blocking `accept4`, edge-triggered read/write readiness, a timer heap, a state
machine per connection reusing `http1` and `buf`. Benchmark goroutine mode, epoll mode,
and nginx against the same backend with `wrk` and `hey`, and profile with
`runtime/pprof`.

**What you will learn.** `epoll(7)` semantics (level vs edge, `EPOLLONESHOT`),
`EAGAIN` handling, partial writes, resumable parsing, and what the Go runtime was doing
for you.

**Reading.** `epoll(7)`, `accept4(2)`, `socket(7)` (`SO_REUSEPORT`), nginx
`ngx_epoll_module.c`, `ngx_event_accept.c`, `ngx_event_timer.c`; Go
`runtime/netpoll_epoll.go`.

**Definition of done.** Same integration suite passes in epoll mode; a table of
requests/s and p99 latency for the three servers at `-c 100`, `-c 1000`, `-c 5000`.

**Effort.** 30 hours, about 6 weeks.

## Timeline

Starting the week of 2026-09-21 at 5 hours per week, with two weeks of buffer over the
holidays:

| Milestone | Hours | Weeks | Done by (approx.) |
|---|---|---|---|
| M1 raw HTTP proxy | 30 | 6 | 2026-11-01 |
| M2 master/worker (or M3 if swapped) | 20 | 4 | 2026-11-29 |
| M3 config + routing (or M2) | 15 | 3 | 2026-12-20 |
| holiday buffer | | 2 | 2027-01-03 |
| M4 timeouts and limits | 10 | 2 | 2027-01-17 |
| M5a TLS + SNI | 12 | 2.5 | 2027-02-03 |
| M5b upstream pooling | 12 | 2.5 | 2027-02-21 |
| **deploy checkpoint** | | | **late February 2027** |
| M6 rate limiting | 8 | 2 | 2027-03-07 |
| M7a balancing + passive health | 10 | 2 | 2027-03-21 |
| M7b active health + retries | 10 | 2 | 2027-04-04 |
| M8a memory cache | 12 | 2.5 | 2027-04-21 |
| M8b disk cache | 12 | 2.5 | 2027-05-09 |
| M9 logs + metrics | 10 | 2 | 2027-05-23 |
| M10 WebSocket | 6 | 1.5 | 2027-06-02 |
| M11 epoll + benchmarks | 30 | 6 | 2027-07-14 |
| **Total** | **207** | **~42** | |

The proxy is usable for your own projects from the deploy checkpoint onward. Milestones
6 to 10 each add one operational capability and can be reordered freely after M5. If
time runs short, M8b, M9's Prometheus output, and M11 are the natural cuts.
