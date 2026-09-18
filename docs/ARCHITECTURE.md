# mini-nginx: Architecture

> Status: Phase 1 design draft. Nothing described here is implemented yet.
> Companion documents: [DOMAIN_MODEL.md](DOMAIN_MODEL.md), [ROADMAP.md](ROADMAP.md),
> [REPO_LAYOUT.md](REPO_LAYOUT.md), [OPEN_DECISIONS.md](OPEN_DECISIONS.md).

## 1. What mini-nginx is

A single static Linux binary that:

- reads an nginx-style configuration file,
- opens listening sockets for HTTP and HTTPS,
- runs a **master** process that supervises **worker** processes and handles signals,
- and, inside each worker, accepts TCP connections, parses HTTP/1.1 by hand, picks a
  virtual host and a location, and proxies the request to a load-balanced,
  health-checked, rate-limited, optionally cached upstream, streaming bytes in both
  directions without buffering whole messages.

Mental model: nginx's `ngx_http_proxy_module` plus just enough of `ngx_http_core_module`
and the master/worker cycle (`ngx_process_cycle.c`) to run it.

Non-goals for now: HTTP/2 and HTTP/3, static file serving, FastCGI/uwsgi, ACME
certificate issuance, forward proxying (`CONNECT`), HTTP trailers, gRPC.

Hard constraints that shape every decision below:

| Constraint | Consequence |
|---|---|
| Standard library only, no `golang.org/x` | `syscall` for Linux specifics, hand-written parsers everywhere |
| No `net/http`, `net/http/httputil`, `net/textproto` on the proxy path | We own the read buffer, the parser, the serializer, and the body framing |
| TLS via `crypto/tls`, but SNI parsed by hand | We peek at the raw ClientHello before handing the connection to `crypto/tls` |
| Linux only | Build tags `//go:build linux` around process and socket code |

## 2. Ten-thousand-foot view

```
                    +---------------------------------------------------------------+
                    |                     mini-nginx (one host)                     |
                    |                                                               |
  client --TCP----->|  :80  --+                                                     |
  client --TLS----->|  :443 --+--> listening sockets, opened ONCE by the master     |
                    |         |                                                     |
                    |         |   +----------+   exec + fd inheritance              |
                    |         +-->|  master  |------------+-----------------+       |
                    |             | no I/O   |            |                 |       |
                    |             | signals  |            v                 v       |
                    |             | config   |     +-------------+   +-------------+|      +-----------+
                    |             | respawn  |     |  worker #1  |   |  worker #2  ||----->| backend A |
                    |             +----------+     | goroutine   |   | goroutine   ||      +-----------+
                    |                              | per conn    |   | per conn    ||      +-----------+
                    |                              +-------------+   +-------------+|----->| backend B |
                    |                                                               |      +-----------+
                    +---------------------------------------------------------------+
```

Three layers, each a separate section below:

1. **Process model** (section 3): master, workers, signals, reload.
2. **Concurrency inside a worker** (section 4): goroutines over the Go netpoller, and
   why a hand-written epoll loop is a later experiment rather than the starting point.
3. **Request lifecycle** (section 5): bytes in, parse, route, upstream, bytes out.

Sections 6 to 9 cover buffers, TLS, the explicit list of differences from nginx, and the
security pitfalls we design against.

## 3. Process model

### 3.1 How nginx does it

The nginx master parses the config, binds the listening sockets, then calls `fork()`
N times. Each child (worker) is a copy of the master: same memory, same parsed config,
same open file descriptors, including the listening sockets. Workers then run a
single-threaded event loop. The master only waits for signals and children.

Analogy: `fork()` is photocopying a running office, staff and open drawers included.
The copies immediately start serving customers using the doors (sockets) that were
already open in the original.

### 3.2 Why Go cannot fork

The Go runtime is multi-threaded from the first instruction: scheduler threads, the
garbage collector, the netpoller. POSIX `fork()` duplicates only the calling thread.
In the child, every other thread simply vanishes mid-instruction, possibly holding a
lock the child will need. That is why the standard library exposes fork only fused
with exec (`syscall.ForkExec`, `os.StartProcess`, `os/exec`): the child never runs Go
code before `exec` replaces it with a fresh program image.

Java has the same limitation (there is no `fork()` in the JVM, only `ProcessBuilder`).
Node's `child_process.fork()` is a misnomer: it spawns a fresh `node` process.

Consequence: our workers are **new processes running the same binary in worker mode**.
Nothing is inherited implicitly. Everything a worker needs must be handed over
explicitly: the listening sockets, the configuration, its identity (generation number).

### 3.3 Option A: re-exec and inherit listening sockets (recommended)

The master opens the listening sockets, then executes its own binary
(`os.Executable()`) with the argument `worker` and passes the sockets through
`exec.Cmd.ExtraFiles`. Entry `i` of `ExtraFiles` becomes file descriptor `3+i` in the
child. The child wraps them with `os.NewFile(3+i, name)` and `net.FileListener(f)`.

```
   master fd table                          worker fd table (after exec)
   ---------------                          ---------------------------
   0  stdin                                 0  /dev/null
   1  stdout                                1  inherited stdout (journal)
   2  stderr                                2  inherited stderr
   3  control socket, master end            3  control socket, worker end   <- ExtraFiles[0]
   4  listen 0.0.0.0:80    ---- same  ----> 4  listen 0.0.0.0:80            <- ExtraFiles[1]
   5  listen 0.0.0.0:443   ---- kernel ---> 5  listen 0.0.0.0:443           <- ExtraFiles[2]
                                object
```

Both processes refer to the **same kernel socket object**, so there is one accept
queue shared by the master (which never calls `accept`) and all workers (which do).
This is exactly how systemd socket activation works (`LISTEN_FDS`, fds starting at 3)
and how nginx's own binary upgrade passes sockets to the new master (the `NGINX`
environment variable lists the fd numbers).

Analogy: fd passing across `exec` is handing a numbered key card to a new employee.
The room (socket) already exists; the card (fd number 4) just lets them in. The worker
does not need to know the address in advance: `net.FileListener` reports `Addr()`
via `getsockname(2)`, and the worker matches that to its config.

Accept behaviour with several workers: all of them wait for connections on the same
socket. Go's netpoller registers the listener with epoll in edge-triggered mode in each
process, so a new connection wakes every worker; one `accept4` succeeds and the others
get `EAGAIN` and go back to sleep. This is the "thundering herd" that nginx addressed
with `accept_mutex` (older) and `EPOLLEXCLUSIVE` (since 1.11.3). We cannot set
`EPOLLEXCLUSIVE` through the netpoller. The cost is `N-1` wasted wakeups per new
connection, negligible for `N <= 4`; milestone 11 measures it.

### 3.4 Option B: `SO_REUSEPORT`

Each worker creates its **own** listening socket bound to the same address with the
`SO_REUSEPORT` option (set from `net.ListenConfig.Control` with
`syscall.SetsockoptInt(fd, SOL_SOCKET, SO_REUSEPORT, 1)`). The kernel hashes the
connection 4-tuple and assigns each incoming connection to exactly one of the sockets.
nginx exposes this as `listen 80 reuseport`.

Analogy: option A is one entrance with several clerks racing to greet each customer.
Option B is the kernel acting as a receptionist who assigns each customer to one of
several separate queues before any clerk sees them.

| | A: inherited socket | B: `SO_REUSEPORT` |
|---|---|---|
| Accept queues | one, shared | one per worker |
| Thundering herd | yes, mild | no |
| Load distribution | first free worker wins (self-balancing) | hash-based, blind to load |
| Reload / worker exit | old worker closes its copy; queue lives on in master; **no connection is lost** | closing a socket **resets connections waiting in its queue** (documented nginx caveat; kernel-side fixes need eBPF steering we cannot use) |
| Who binds | master (privileged once) | every worker |
| Matches nginx default | yes | opt-in in nginx too |

**Recommendation:** A as the default. B becomes an opt-in `reuseport` flag on `listen`
in milestone 11, where we benchmark both.

### 3.5 Master responsibilities and signals

The master never touches client bytes. It:

1. parses and validates the config (also behind `mini-nginx -t`),
2. opens listening sockets (privileged ports need `CAP_NET_BIND_SERVICE`, see
   [OPEN_DECISIONS.md](OPEN_DECISIONS.md)),
3. spawns `worker_processes` workers with the current config **generation**,
4. writes the pid file,
5. loops on signals and child exits.

| Signal to master | nginx | mini-nginx |
|---|---|---|
| `SIGHUP` | reload configuration | same (section 3.7) |
| `SIGQUIT` | graceful shutdown | same |
| `SIGTERM`, `SIGINT` | fast shutdown | graceful with a deadline, then `SIGKILL` (systemd sends `TERM`; see open decisions) |
| `SIGUSR1` | reopen log files | same (milestone 9) |
| `SIGUSR2` | binary upgrade | not planned; the mechanism is the same fd inheritance |
| `SIGWINCH` | gracefully stop workers | not planned |
| `SIGCHLD` | reap, respawn workers | same |

| Signal to worker | meaning |
|---|---|
| `SIGQUIT` | drain: stop accepting, finish in-flight requests, close idle keep-alives, exit |
| `SIGTERM` | exit now |
| `SIGUSR1` | reopen log files |

Command line mirrors nginx: `mini-nginx -c /etc/mini-nginx.conf`, `-t` (test config),
`-s reload|quit|stop|reopen` (reads the pid file and sends the matching signal).

### 3.6 Worker lifecycle and the control channel

Spawning one worker:

```
  master                                              worker (same binary, arg "worker")
    |
    | 1. socketpair(AF_UNIX, SOCK_STREAM) -> (m, w)
    | 2. exec self: ExtraFiles=[w, listeners...],
    |    env MININGINX_GENERATION=<n>,
    |    SysProcAttr{Pdeathsig: SIGTERM}              ---->  3. fd 3 = control, fd 4.. = listeners
    | 4. send frame: [u32 len][config bytes]          ---->  5. parse config with the SAME parser
    |                                                        6. net.FileListener(fd 4..), build routers,
    |                                                           start accept loops
    |                                             <----       7. send frame "READY"
    | 8. mark worker Running
    |                                                        9. blocking read on fd 3 forever:
    |                                                           EOF => master is gone => drain and exit
```

Why send the config **bytes** and not the path: the master validated *these* bytes. If
the worker re-read the file it might see a newer, possibly broken, version. nginx does
not have this problem because `fork()` copies the already-parsed config.

Why not serialize the parsed config (`encoding/gob`): it would drag along compiled
regexes, TLS material and runtime-only fields. Re-parsing identical bytes with a
deterministic parser is cheap and keeps one source of truth. The same channel carries
anything else the master validated on the worker's behalf (later: certificate PEM
bytes, so only the master needs read access to private keys).

Master-death detection is done twice: `Pdeathsig` (kernel sends a signal when the
parent **thread** exits, which in Go is not always the parent process, so it is not
sufficient alone) and EOF on the control socket (reliable). nginx uses a `socketpair`
channel per worker for the same purpose (`ngx_channel.c`), and additionally passes fds
over it with `SCM_RIGHTS` (`syscall.UnixRights` in Go); we keep that as an optional
exercise, since re-spawning workers on reload makes it unnecessary.

### 3.7 Graceful reload (`SIGHUP`)

```
  admin         master                      old workers (gen 1)         new workers (gen 2)
    |  HUP        |                               |                            |
    |------------>| parse + validate new config   |                            |
    |             |   error? log it, keep gen 1,  |                            |
    |             |   DONE (nginx does the same)  |                            |
    |             | open listeners that are new;  |                            |
    |             | keep the ones that still exist|                            |
    |             |-------------------- exec, send config, listeners --------->|
    |             |<------------------------------------------------ READY ----|
    |             |   (no READY within deadline? kill gen 2, keep gen 1)       |
    |             |---- SIGQUIT ----------------->|                            |
    |             |                               | close own listener fds     |
    |             |                               | (queue lives on in master) |
    |             |                               | stop accepting             |
    |             |                               | close idle keep-alive conns|
    |             |                               | finish in-flight requests  |
    |             |                               | with "Connection: close"   |
    |             |<--- exit (SIGCHLD) -----------|                            |
    |             | close listeners removed from config                        |
    |             | rewrite pid bookkeeping; gen 2 is now current              |
```

Two deliberate differences from nginx: we wait for `READY` before draining the old
generation (nginx sends `QUIT` immediately after `fork`), and we roll back if the new
workers never become ready. Both make a botched reload harmless.

Draining is the nginx behaviour of `ngx_exiting`: an idle keep-alive connection is
closed at once; a connection in the middle of a request finishes that response with
`Connection: close`. `worker_shutdown_timeout` bounds the wait, after which the master
sends `SIGKILL`.

### 3.8 Graceful shutdown and crash recovery

- `SIGQUIT` to master: `SIGQUIT` to every worker, wait up to `worker_shutdown_timeout`,
  `SIGKILL` stragglers, remove the pid file, exit.
- `SIGTERM`: same sequence with a shorter deadline (open decision).
- A worker that exits unexpectedly is respawned with the same generation and config.
  Respawns are rate-limited (nginx sleeps a second when children die too fast, see
  `ngx_reap_children`) so a crash loop does not spin the CPU.

### 3.9 Why keep workers at all in Go?

One Go process already runs goroutines on every core (`GOMAXPROCS` defaults to the CPU
count). nginx needs N workers because each is single-threaded; we do not. We keep the
model anyway because:

1. **Isolation.** A panic or a runaway goroutine kills one worker's connections, not
   all of them. The master respawns it.
2. **Reload semantics.** "New process with new config, old process drains" is the
   cleanest possible reload; there is no shared mutable state to migrate.
3. **Privilege separation.** The master can hold privileged resources; workers cannot.
4. **It is the point of the project.**

Default `worker_processes 1`, which is also nginx's default. `auto` maps to CPU count.

## 4. Concurrency inside a worker

### 4.1 nginx: one thread, one epoll loop, state machines

Analogy: a doorbell panel. Instead of walking to each door asking "anyone there?", the
worker registers every socket with epoll and then blocks in `epoll_wait`. The kernel
returns the list of sockets that have something to do. The worker performs the next
small non-blocking step for each (read a few more header bytes, write a few more
response bytes) and goes back to waiting. It never blocks on a single socket.
Java's NIO `Selector` and Node's libuv loop are the same idea.

The price: every operation must be written as a resumable state machine.
`ngx_http_parse_request_line` alone has around forty states because it may be
interrupted after any byte.

### 4.2 Go: goroutine per connection over the netpoller

Go's runtime already contains an epoll loop (`runtime/netpoll_epoll.go`). When a
goroutine calls `conn.Read` and no bytes are available, the runtime parks the goroutine
(a few KiB of stack), registers interest with epoll, and runs other goroutines on the
same OS thread. When epoll reports readiness, the goroutine is resumed where it left
off. You write blocking-style code and get event-driven execution.

Analogies: Java virtual threads (Project Loom) do exactly this. In Node terms, it is as
if every `await socket.read()` were free and the event loop were invisible.

### 4.3 Trade-off and recommendation

| | (a) goroutine per connection | (b) hand-written epoll loop |
|---|---|---|
| Code shape | sequential, readable | state machines, callbacks |
| Memory per idle connection | goroutine stack (starts at 8 KiB in recent Go) + our buffers | our buffers + a small state struct |
| Blocking calls | harmless, runtime parks them | any blocking call stalls every connection |
| Timeouts | `SetReadDeadline` / `SetWriteDeadline` | a timer heap you write yourself |
| TLS | `crypto/tls` wraps a blocking `net.Conn`; works | `crypto/tls` does not fit a non-blocking loop; you would end up with a goroutine per TLS connection anyway |
| Learning value | Go idioms, netpoller internals | epoll(7), nginx internals, accept4, EPOLLET |
| Throughput | very good | possibly better at very high connection counts; must be measured |

**Recommendation:** start with (a). Make (b) milestone 11, restricted to plaintext
HTTP, as a measured experiment against both (a) and real nginx.

Design constraint we keep from day one so that (b) stays possible: the HTTP parser
operates on a `[]byte` buffer and returns `ErrNeedMore` when the head is incomplete. It
never takes an `io.Reader`. Body readers pull from a buffer plus a source. That is the
shape nginx uses, and it is the shape an event loop needs.

### 4.4 Goroutines per connection, concretely

- One goroutine per client connection runs the keep-alive loop (section 5.8).
- During an exchange, the same goroutine writes the request head, streams the request
  body to the upstream, then reads and streams the response. This is sequential in
  milestone 1. Simplification to note: if the upstream replies **before** it has read
  the whole request body (for example `413`), we notice only after the upload ends or
  fails. Milestone 7 moves the body upload to a second goroutine so the response can be
  read concurrently, which is what nginx's event loop gets for free.
- WebSocket tunnels (milestone 10) use two goroutines, one per direction.
- Active health checks, cache sweeps and rate-limit bucket expiry run as a handful of
  background goroutines per worker.

## 5. Request lifecycle: bytes in to bytes out

```
 client                 ClientConn goroutine (one worker)                        upstream
   |                        |                                                       |
   |== TCP bytes ==========>| read() into readBuf                                   |
   |                        | scan for CRLF CRLF; parse request line + field lines  |
   |                        | decide body framing (Content-Length / chunked / none) |
   |                        | validate: Host, smuggling rules, sizes                |
   |                        | route: Listener -> VirtualHost (Host) -> Route (path) |
   |                        | gates: max conns, rate limit (M6), cache lookup (M8)  |
   |                        | pick Backend (M7), get UpstreamConn (pool M5 / dial)  |
   |                        | serialize canonical request head =====================>
   |== body bytes =========>| stream body: CL = n bytes; chunked = decode+re-encode ==>
   |                        |<===================== read + parse response head ======
   |<== response head ======| strip hop-by-hop, decide client framing, serialize    |
   |<== body bytes =========| stream body: CL / chunked / until-close (re-chunked) <==
   |                        | release UpstreamConn (pool or close)                  |
   |                        | write access log line (M9)                            |
   |                        | keep-alive? -> back to read() : close                 |
```

### 5.1 Accept

`net.Listener.Accept` returns a `*net.TCPConn` (`TCP_NODELAY` is on by default in Go,
which matches nginx's `tcp_nodelay on`). The worker checks the connection gate
(`worker_connections`), takes a `ClientConn` from a pool, and starts its goroutine.
nginx equivalent: `ngx_event_accept` followed by `ngx_http_init_connection`.

### 5.2 Reading the head

The connection owns one fixed-size read buffer (8 KiB by default). We read into it
until the buffer contains `\r\n\r\n`, scanning only the bytes that arrived since the
last scan, or until the buffer is full, which yields `431 Request Header Fields Too
Large`. nginx: `client_header_buffer_size 1k` plus `large_client_header_buffers 4 8k`;
we use one buffer whose capacity is the header size limit, with no "grow" step.

Real bytes, one `read()` of 62 bytes:

```
0000  47 45 54 20 2f 61 70 69 2f 76 31 20 48 54 54 50   GET /api/v1 HTTP
0010  2f 31 2e 31 0d 0a 48 6f 73 74 3a 20 61 2e 74 65   /1.1..Host: a.te
0020  73 74 0d 0a 43 6f 6e 74 65 6e 74 2d 4c 65 6e 67   st..Content-Leng
0030  74 68 3a 20 35 0d 0a 0d 0a 68 65 6c 6c 6f         th: 5....hello
```

The body `hello` arrived in the same read. The parser therefore returns how many bytes
it consumed (57 here) so the remaining 5 bytes are handed to the body reader instead of
being lost. The same rule makes pipelined requests work (section 5.8).

Parsing rules, all from RFC 9112, and what we do on violation:

| Rule | RFC 9112 | Action |
|---|---|---|
| `request-line = method SP request-target SP HTTP-version CRLF`, method is a `token` | §3 | 400 |
| Version must be `HTTP/1.1` or `HTTP/1.0` | §2.3 | 505 for other 1.x, 400 otherwise |
| Request target forms: origin (`/p?q`), absolute (`http://h/p`), authority (CONNECT only), asterisk (`OPTIONS *`) | §3.2 | accept origin and absolute; 400 for authority (no CONNECT); asterisk only for OPTIONS |
| Exactly one `Host` for HTTP/1.1; must agree with absolute-form authority | §3.2 | 400 |
| No whitespace between field name and colon | §5.1 | 400 |
| Obsolete line folding (line starts with SP/HTAB) | §5.2 | 400 |
| Bare CR, or LF without CR | §2.2 | 400 (stricter than nginx; open decision) |
| Field values contain no CR, LF, NUL (we also reject other C0 controls) | RFC 9110 §5.5 | 400 |
| Field name is a `token` | §5 | 400 |

### 5.3 Body framing and the request-smuggling table

Analogy: request smuggling is two clerks (the proxy and the backend) disagreeing about
where letter 1 ends and letter 2 begins in the same envelope stream. The second clerk
"reads" a letter the first never saw, or vice versa.

Decision table for requests (RFC 9112 §6.3, nginx behaviour matches):

| Present | Framing | Notes |
|---|---|---|
| `Transfer-Encoding` **and** `Content-Length` | 400, close | RFC allows dropping CL; we refuse, like nginx |
| `Transfer-Encoding: chunked` (chunked is the last coding) | chunked | |
| `Transfer-Encoding` with any other coding, or chunked not last | 400 (501 for unknown codings) | §6.1 |
| Single `Content-Length: digits` | fixed, N bytes | leading `+`, spaces, non-digits: 400 |
| Repeated `Content-Length` with identical values | fixed | §6.3 item 5 |
| Repeated or list-valued `Content-Length` with differing values | 400, close | |
| Neither | no body | requests never default to "until close" |

Responses add: any response to `HEAD`, and any `1xx`, `204`, `304`, has no body
regardless of headers; a response with neither header is delimited by connection close.

Our structural defense goes further than the table: **we never forward the client's raw
bytes**. We parse, decide the framing, and re-serialize a canonical request with exactly
one framing header and our own chunk encoding. The backend cannot disagree with us
because it never sees the ambiguous input. nginx does the same in
`ngx_http_proxy_create_request`.

### 5.4 Routing

1. The listener the connection arrived on holds an ordered list of virtual hosts and a
   default. The `Host` header (lowercased, port stripped) is matched exactly, then
   against wildcards (`*.example.com`), then falls back to the default server. For TLS,
   SNI selected the certificate; `Host` still selects the server block, exactly like
   nginx. A mismatch is allowed (nginx allows it too); `421 Misdirected Request` is an
   option we keep in reserve.
2. Inside the virtual host, locations are matched nginx-style: `=` exact first, then
   the longest prefix, `^~` stopping the search, regex locations (`~`, `~*`) as a later
   step. The matched `Route` already carries its *effective* settings (timeouts,
   limits, headers), merged at config load time the way nginx merges location
   configurations (`ngx_http_core_merge_loc_conf`).
3. The route's action is `proxy_pass` to an upstream, or `return <status> [text]`.

### 5.5 Building the upstream request

Header rewrite, in order (nginx names in parentheses):

1. **Drop hop-by-hop headers** (RFC 9110 §7.6.1): `Connection` and every header it
   names; plus by convention `Keep-Alive`, `Proxy-Connection`, `TE`, `Trailer`,
   `Transfer-Encoding` (we write our own), `Upgrade` (until milestone 10),
   `Proxy-Authorization`, `Proxy-Authenticate`.
2. **Forwarding headers** (`proxy_set_header`): append the client IP to
   `X-Forwarded-For` (`$proxy_add_x_forwarded_for`), set `X-Forwarded-Proto`
   (`http`/`https`), set `X-Forwarded-Host` to the original `Host`, set `X-Real-IP`.
   We append rather than replace because backends read the **last** value; a
   `trusted_proxies` list (later) decides whether the incoming value is believed at all.
3. **`Host`**: forwarded unchanged (open decision; nginx's default is `$proxy_host`).
4. **`Connection: close`** in milestone 1 (one TCP connection per request) and
   `Connection: keep-alive` once pooling exists (milestone 5). nginx defaults to
   `proxy_http_version 1.0` and `Connection close`; we always speak HTTP/1.1 upstream.
5. **Request target** in origin-form; absolute-form from the client is converted.
   URI rewriting for `proxy_pass http://up/prefix/` follows nginx's replace-the-matched-
   prefix rule (milestone 3).
6. **Header injection defense**: the serializer refuses any name or value containing
   CR, LF or NUL even though the parser already rejected them. Values we synthesize
   (`X-Forwarded-For`) are built from validated parts only.

Example, request as received and as sent:

```
client -> mini-nginx                        mini-nginx -> backend
GET /api HTTP/1.1                           GET /api HTTP/1.1
Host: a.test                                Host: a.test
Connection: keep-alive, X-Debug             User-Agent: curl/8.7
X-Debug: 1                                  Accept: */*
Keep-Alive: timeout=5                       X-Forwarded-For: 203.0.113.9
User-Agent: curl/8.7                        X-Forwarded-Proto: http
Accept: */*                                 X-Forwarded-Host: a.test
                                            Connection: close
```

`X-Debug` disappears because the client listed it in `Connection`.

### 5.6 Streaming the request body

Never allocate a buffer the size of the body.

- **Content-Length N**: hand the body reader the leftover bytes already in `readBuf`,
  then read from the socket until exactly N bytes have been copied, through a pooled
  32 KiB copy buffer. Reading one byte past N would eat the next pipelined request.
- **Chunked** (RFC 9112 §7.1): read `chunk-size [;ext] CRLF`, copy that many bytes,
  expect `CRLF`, repeat; a size of `0` ends the body, followed by an optional trailer
  section and a final `CRLF`. Chunk sizes are hex, capped to guard against overflow.
  Chunk extensions are ignored. Trailers are consumed and dropped in milestone 1.
  We re-encode to the upstream with our own `ChunkedWriter`; chunk boundaries may differ
  from the client's, which is legal.

```
  "5\r\nhello\r\n6\r\n world\r\n0\r\n\r\n"
   ^^^^  ^^^^^  ^^^^ ^^^^^^^^^^^^  ^^^^ ^^^^
   size  data   crlf size+data...  last empty trailer section
```

Analogy: chunked encoding is mailing a book chapter by chapter. Each envelope says how
many pages are inside; an empty envelope labelled `0` means "the end", and it may
carry a P.S. (trailers).

- `client_max_body_size` (milestone 4) is enforced while counting, for both framings.
- `Expect: 100-continue` (RFC 9110 §10.1.1): forwarded to the upstream, and the
  upstream's `100 Continue` is relayed. Milestone 1 stretch goal.

### 5.7 Response path

1. Read the upstream head into the upstream connection's buffer (nginx:
   `proxy_buffer_size`), parse the status line and fields with the same field parser.
2. Decide the upstream body framing (section 5.3, response rules).
3. Rewrite headers: drop hop-by-hop; set `Connection` for the client according to
   section 5.8; add `X-Cache-Status` later.
4. Decide the **client** framing: if the upstream body is length-delimited or chunked,
   keep it. If it is close-delimited and the client speaks HTTP/1.1, **re-chunk** it so
   the client connection can stay alive (nginx: `ngx_http_chunked_filter_module`). For
   an HTTP/1.0 client, send close-delimited with `Connection: close`.
5. `1xx` responses are forwarded to HTTP/1.1 clients and we keep reading for the final
   response (RFC 9110 §15.2). `HEAD`, `204`, `304`: write the head only.
6. Stream the body through a pooled buffer. We do not buffer the response to free the
   upstream early (nginx `proxy_buffering on` does); a slow client therefore holds a
   backend connection for the duration. Acceptable for personal projects; noted as a
   difference in section 8.
7. Release the upstream connection: back to the pool if the response was fully read and
   both sides allow reuse, otherwise close.

### 5.8 Client keep-alive, pipelining, and connection state

```
   accept
     |
     v
  +----------+  head complete  +-----------+  proxied   +----------+
  | ReadHead |---------------->| Exchange  |----------->| WriteEnd |
  +----------+                 +-----------+            +----------+
     ^   |                                                  |
     |   | parse error / timeout / too large                | keep-alive allowed
     |   v                                                  | AND body fully consumed
  +--------+                                                | AND not draining
  | Error  |--> write 4xx/5xx with Connection: close        v
  +--------+                                            +-------+
     |                                                  | Idle  |--- idle timeout ---> close
     v                                                  +-------+
   close                                                    |  leftover bytes or new read
                                                            +------------> ReadHead
```

Keep-alive decision: HTTP/1.1 defaults to persistent unless `Connection: close` is
present; HTTP/1.0 defaults to close unless `Connection: keep-alive`. We also close after
`keepalive_requests` requests, when draining, or whenever we could not be sure where the
next request begins (any 400, an unread body).

Pipelining: leftover bytes after one request are simply the beginning of the next.
Because one goroutine processes requests sequentially, responses go out in order, which
is all RFC 9112 §9.3.2 requires.

### 5.9 Locally generated responses

| Status | When |
|---|---|
| 400 | any parsing or framing violation |
| 408 | header or body read timeout (milestone 4) |
| 413 | body exceeds `client_max_body_size` |
| 431 | head does not fit the read buffer |
| 429 | rate limited (nginx uses 503; open decision) |
| 502 | upstream refused, reset, or sent an unparseable response |
| 503 | no live backend, connection limit, draining |
| 504 | upstream connect or read timeout |

Each is a fixed text body with `Content-Length`; error responses to malformed input
carry `Connection: close` because the byte stream is no longer trustworthy.

## 6. Buffer handling

nginx allocates fixed-size buffers up front and moves pointers through them
(`ngx_buf_t` has `start`, `pos`, `last`, `end`). We do the same with one small type:

```
   readBuf (cap 8192)
   +----------------------+----------------------+----------------------+
   | consumed (parsed)    | unread               | free                 |
   +----------------------+----------------------+----------------------+
   0                      r                      w                     cap

   After parsing a head, bytes [r:w) are body bytes or the next pipelined request.
   Before reading more: if r > 0 and w == cap, memmove [r:w) to 0 (compaction);
   then read into [w:cap).
```

| nginx directive | mini-nginx |
|---|---|
| `client_header_buffer_size`, `large_client_header_buffers` | `ClientConn.readBuf`, one fixed buffer; its capacity is the header limit |
| `proxy_buffer_size` | `UpstreamConn.readBuf` for the response head |
| `proxy_buffers`, `proxy_buffering` | none; bodies stream through a pooled copy buffer |
| `client_body_buffer_size`, `proxy_request_buffering` | none; request bodies stream |

Ownership rules:

- A `ClientConn` owns its read buffer for the life of the connection. Body readers
  borrow it and must not retain slices past the exchange.
- Copy buffers (32 KiB) come from a `sync.Pool` and go back in a `defer`.
- Header names and values are copied into Go strings in v1. nginx keeps `ngx_str_t`
  views into the buffer instead; zero-copy views are a milestone 11 optimization.
- Nothing allocated during an exchange outlives it except the access-log line.

Analogy: Netty's pooled `ByteBuf` allocator, or a stack of reusable trays in a canteen.

Zero-copy note: `net.TCPConn.ReadFrom` uses `splice(2)` on Linux when the source is a
`*net.TCPConn` or an `io.LimitedReader` over one. Content-Length bodies and WebSocket
tunnels can hand the copy to the kernel. Optional, measured in milestones 10 and 11.

## 7. TLS termination in one picture

```
  client ----ClientHello---->  worker
                                 | 1. read the first TLS record(s) into a buffer
                                 | 2. hand-parse: record header, handshake header,
                                 |    ClientHello fields, extensions, server_name (SNI)
                                 | 3. CertStore.Lookup(sni) -> *tls.Certificate
                                 | 4. conn' = prefixConn{peeked bytes + net.Conn}
                                 | 5. tls.Server(conn', &tls.Config{Certificates: [cert],
                                 |                                   NextProtos: ["http/1.1"]})
                                 | 6. Handshake(); then treat tls.Conn exactly like a
                                 |    plaintext conn in section 5
```

`crypto/tls` could give us the SNI via `Config.GetCertificate(*ClientHelloInfo)`, and we
will use that callback as a cross-check in tests. The hand parser is the learning goal:
a TLS record is `type(1) version(2) length(2)`, a handshake message is
`type(1) length(3)`, and the ClientHello is a sequence of length-prefixed vectors
(RFC 8446 §4.1.2). A ClientHello may span several records; the parser accumulates
handshake bytes until the declared handshake length is satisfied.

Analogy: the TLS handshake is two people exchanging business cards in public and then
agreeing on a secret language. SNI is the client saying which company it came to see
*before* any cards are exchanged, so the receptionist knows which card to hand over.

Bytes of a real ClientHello, annotated (details in milestone 5):

```
16 03 01 02 00           record: type 0x16 handshake, legacy version 3.1, length 0x0200
01 00 01 fc              handshake: type 0x01 ClientHello, length 0x0001fc
03 03                    legacy_version TLS 1.2 (1.3 hides in supported_versions)
<32 bytes>               random
20 <32 bytes>            legacy_session_id, length 0x20
00 3e <62 bytes>         cipher_suites, length 0x3e
01 00                    compression_methods: 1 entry, null
01 75                    extensions, total length 0x0175
   00 00 00 0b           ext type 0x0000 server_name, length 11
      00 09              ServerNameList length 9
      00                 name_type host_name
      00 06 61 2e 74 65 73 74     length 6, "a.test"
   00 0a ...             supported_groups, and so on
```

## 8. Where mini-nginx differs from nginx, and why

| Topic | nginx | mini-nginx | Why |
|---|---|---|---|
| Worker creation | `fork()` | re-exec + fd inheritance | Go runtime cannot fork |
| Config handoff | inherited memory | bytes over a control socket, re-parsed | same reason |
| Worker concurrency | single-threaded epoll loop | goroutines on the runtime netpoller | epoll is already there; state machines cost more than they teach at first |
| Accept contention | `accept_mutex` / `EPOLLEXCLUSIVE` | none, tolerate spurious wakeups | not reachable through the netpoller |
| Shared state between workers | shared memory zones (`limit_req_zone`, upstream `zone`, cache index) | per-worker state; default one worker | `syscall.Mmap` plus atomics is possible but is a project of its own |
| Cache manager / loader | separate processes | goroutines in the worker | simpler; one worker by default |
| Response buffering | `proxy_buffering on` | streaming only | learning goal is byte streaming; slow clients hold backends |
| Request body | buffered to memory/disk by default | streamed | same; limits retries after body bytes were sent |
| Upstream HTTP version | 1.0 by default | always 1.1 | keep-alive and chunked upstream bodies |
| `Host` to upstream | `$proxy_host` | client `Host` unchanged | open decision |
| Regex engine | PCRE | `regexp` (RE2 semantics, no backtracking) | stdlib only; also immune to ReDoS |
| Lenient line endings | accepts bare LF | rejects | simpler parser, safe because we re-serialize; open decision |
| Reload readiness | `QUIT` old workers right after `fork` | wait for `READY`, roll back on failure | cheap safety |
| HTTP/2 | yes | no | out of scope |

## 9. Security pitfalls we design against

- **Request smuggling**: framing decision table (5.3), re-serialization, close after
  any 400. Tested with raw `nc` fixtures from milestone 1 on.
- **Response splitting / header injection**: parser and serializer both reject CR, LF,
  NUL in names and values; synthesized headers are built from validated parts.
- **Oversized input**: fixed read buffer (431), `client_max_body_size` (413), capped
  chunk sizes, capped number of header fields, capped URI length.
- **Slow clients (Slowloris)**: `client_header_timeout`, `client_body_timeout`,
  `keepalive_timeout`, `send_timeout`, `worker_connections` gate (milestone 4).
- **Slow or dead backends**: `proxy_connect_timeout`, `proxy_read_timeout`,
  `proxy_send_timeout`, passive health marking (milestones 4 and 7).
- **Spoofed `X-Forwarded-For`**: append, never trust blindly; `trusted_proxies` later.
- **Retry safety**: retry only idempotent methods and only before any body byte was
  sent or before any response byte was received (RFC 9112 §9.3.1).
- **TLS**: ALPN advertises `http/1.1` only; minimum version TLS 1.2; certificate
  selection never falls through to a wrong host's key without an explicit default.
- **Rate limiting** keyed by client IP, with a bounded bucket table.
