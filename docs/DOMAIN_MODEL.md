# mini-nginx: Domain Model

> Status: Phase 1 design draft. Go sketches below show fields and signatures only.
> Names may still change during review; responsibilities should not.

## 1. How to read this document

- Types are grouped by **bounded context** (process, config, wire, routing, proxy, ...).
- Each type has a one-sentence responsibility and the milestone `[Mn]` that introduces it.
- Two rules apply everywhere:
  1. **Config types are immutable values** after loading. They describe intent.
  2. **Runtime types own mutable state** and hold a pointer to the config value they
     realize. Java analogy: config types are records/DTOs, runtime types are services.
     nginx blurs the two (its config structs carry runtime fields); we separate them so
     reload is trivially safe: a new generation gets new runtime objects.

## 2. Relationship map

```
  PROCESS                                         one binary, two roles
  =======
  Master 1 --spawns--> * WorkerHandle  ~~~ (OS process boundary) ~~~  Worker 1
    | owns                                                             | owns
    v                                                                  v
  config.Config  ----------- same bytes, parsed twice -----------> config.Config
    |                                                                  |
    | 1..*                                                             | builds runtime for
    v                                                                  v
  CONFIG (immutable)                                             RUNTIME (per worker)
  ==================                                             ===================
  Listener 1 --- * VirtualHost 1 --- * Route --- 1 Action        ListenerRT 1 --- 1 Router
     |                |                 |          |                 |
     | tls            | limits          | limits   | proxy_pass      | accepts
     v                v                 v          v                 v
  TLSSettings       Limits            Limits    Upstream 1 -- * Backend     ClientConn *
                                        |                                      | per request
                                        | M6 / M8                              v
                                        v                                  Exchange 1
                                RateRule / CacheRule                           |
                                                                    uses ------+------ uses
                                                                    |                 |
                                                                    v                 v
                                          upstream.Pool 1 -- * Peer -- * Conn   http1.RequestHead
                                                |                (idle list)   http1.ResponseHead
                                                | Balancer, Prober, RetryPolicy  http1.BodyReader
                                                                                http1.ChunkedWriter
  CROSS-CUTTING per worker: buf.Pool, ratelimit.Zone, cache.Store, accesslog.Logger,
                            metrics.Counters, limits.ConnGate, tlsx.CertStore
```

## 3. Process context (package `master`, `worker`, `proc`)

| Type | Responsibility | Milestone |
|---|---|---|
| `master.Master` | Owns the config file path, the validated config bytes, the listening sockets, and the set of workers; runs the signal loop; never serves traffic. | M2 |
| `master.WorkerHandle` | Master-side view of one worker process: pid, generation, control socket, state (`Starting`, `Running`, `Draining`, `Exited`). | M2 |
| `master.Generation` | Integer stamped on every worker; a reload creates generation `n+1` and drains generation `n`. | M2 |
| `worker.Worker` | Worker-side root: parses the config it received, builds one `ListenerRT` per inherited socket, tracks live connections, implements drain. | M2 |
| `worker.ListenerRT` | Runtime for one listening socket: the `net.Listener`, its `Router`, its `CertStore`, and the accept loop. | M1 (single, hard-coded), M2 (inherited), M3 (from config) |
| `proc.ControlConn` | Length-prefixed message framing over the `socketpair` between master and worker (`CONFIG`, `READY`, later `REOPEN`). | M2 |
| `proc.Inherited` | Linux-only helpers: build `ExtraFiles`, recover fds 3.. in the child, set `Pdeathsig`, optional `Credential`, optional `SO_REUSEPORT`. | M2, M11 |

```go
// package master  (linux only)
type Master struct {
    cfgPath   string
    cfgText   []byte            // validated bytes of the current generation
    cfg       *config.Config
    listeners map[string]*os.File // key: "0.0.0.0:80"; opened once, shared by generations
    gen       Generation
    workers   map[int]*WorkerHandle // by pid
    sigs      chan os.Signal
    pidFile   string
}

type WorkerHandle struct {
    PID     int
    Gen     Generation
    Ctl     *proc.ControlConn
    State   WorkerState
    Started time.Time
}

// package worker
type Worker struct {
    cfg       *config.Config
    gen       master.Generation
    listeners []*ListenerRT
    ctl       *proc.ControlConn
    draining  atomic.Bool
    conns     *limits.ConnGate
    live      sync.WaitGroup      // one Add per ClientConn
    pools     map[string]*upstream.Pool
    zones     map[string]*ratelimit.Zone // M6
    cache     cache.Store               // M8
    access    *accesslog.Logger         // M9
    metrics   *metrics.Counters         // M9
}

type ListenerRT struct {
    cfg    *config.Listener
    ln     net.Listener
    router *router.Router
    certs  *tlsx.CertStore // nil for plaintext
}
```

## 4. Configuration context (package `config`)

| Type | Responsibility | Milestone |
|---|---|---|
| `config.Directive` | One node of the parsed config tree: name, arguments, optional block, source position (file, line, column). This is the raw AST, like nginx's `ngx_conf_t` stream of directives. | M3 |
| `config.Config` | Root of the typed configuration: global settings plus listeners, virtual hosts, upstreams. Produced by the schema step from the AST. | M3 (M1 uses a hard-coded instance) |
| `config.Listener` | One `listen` address: `addr:port`, TLS on/off, `reuseport`, backlog, the virtual hosts that listen there, and the default one. Derived by grouping `listen` directives across `server` blocks, as nginx builds `ngx_http_conf_addr_t`. | M3 |
| `config.VirtualHost` | One `server` block: names (exact and `*.` wildcards), TLS material, ordered routes, effective limits. Matched by `Host` (and SNI for the certificate). | M3 |
| `config.Route` | One `location`: a `Matcher`, an `Action`, and the effective (merged) per-location settings: limits, rate-limit rule, cache rule, header rules. | M3 |
| `config.Matcher` | How a path is matched: `Exact` (`=`), `Prefix`, `PrefixNoRegex` (`^~`), `Regex` (`~`, `~*`). | M3 |
| `config.Action` | What a route does: `ProxyAction{Upstream, URIRewrite, HeaderRules, PassHost}` or `ReturnAction{Status, Body|Location}`. | M3 |
| `config.Upstream` | An `upstream` block: name, backends, balancing policy, `keepalive` idle pool size, TLS-to-backend settings, health-check and retry policy. | M3 (single backend in M1) |
| `config.Backend` | One `server` line inside `upstream`: address, `weight`, `max_fails`, `fail_timeout`, `backup`, `down`. | M3 |
| `config.Limits` | All timeouts and sizes that can be set globally, per server, or per location, already merged. | M4 |
| `config.TLSSettings` | Certificate and key paths, minimum version, whether to redirect plaintext. | M5 |

```go
type Config struct {
    WorkerProcesses int
    ShutdownTimeout time.Duration   // worker_shutdown_timeout
    PIDFile         string
    ErrorLog        string
    AccessLog       *AccessLogSpec  // M9
    Listeners       []*Listener
    Servers         []*VirtualHost
    Upstreams       map[string]*Upstream
    RateZones       map[string]*RateZone // M6
    CachePaths      map[string]*CachePath // M8
    Limits          Limits          // global defaults
}

type Listener struct {
    Addr      string        // "0.0.0.0:80", "[::]:443"
    TLS       bool
    ReusePort bool          // M11
    Backlog   int
    Servers   []*VirtualHost
    Default   *VirtualHost
}

type VirtualHost struct {
    Names    []string      // lowercased; "*.example.com" allowed
    TLS      *TLSSettings  // M5
    Routes   []*Route      // exact first, then prefixes longest-first, then regex in file order
    Limits   Limits
}

type Route struct {
    Match     Matcher
    Action    Action
    Limits    Limits
    RateLimit *RateRule   // M6
    Cache     *CacheRule  // M8
}

type Matcher struct {
    Kind MatchKind        // MatchExact, MatchPrefix, MatchPrefixStop, MatchRegex, MatchRegexCI
    Path string
    Re   *regexp.Regexp   // stdlib RE2; nil unless Kind is regex
}

type ProxyAction struct {
    Upstream  *Upstream
    URIPrefix string       // replacement for the matched prefix, "" = pass through
    Headers   []HeaderRule // proxy_set_header
    PassHost  bool         // forward client Host (true) or use upstream name
}

type Upstream struct {
    Name      string
    Backends  []*Backend
    Policy    BalancePolicy      // RoundRobin (default), LeastConn, IPHash
    KeepAlive int                // idle connections to keep per worker (0 = none)
    TLS       *UpstreamTLS       // M5: proxy_pass https://, verify, server name
    Health    *ActiveCheck       // M7: path, interval, thresholds
    Retry     RetryPolicy        // M7: proxy_next_upstream
}

type Backend struct {
    Addr        string
    Weight      int
    MaxFails    int
    FailTimeout time.Duration
    Backup      bool
    Down        bool
}

type Limits struct {
    HeaderTimeout   time.Duration // client_header_timeout
    BodyTimeout     time.Duration // client_body_timeout (between reads)
    IdleTimeout     time.Duration // keepalive_timeout
    SendTimeout     time.Duration // send_timeout (between writes)
    ConnectTimeout  time.Duration // proxy_connect_timeout
    UpstreamRead    time.Duration // proxy_read_timeout
    UpstreamSend    time.Duration // proxy_send_timeout
    MaxHeaderBytes  int           // capacity of the read buffer
    MaxBodyBytes    int64         // client_max_body_size, 0 = unlimited
    KeepAliveReqs   int           // keepalive_requests
}
```

## 5. Wire context: buffers and HTTP/1.1 (packages `buf`, `http1`)

| Type | Responsibility | Milestone |
|---|---|---|
| `buf.Buffer` | A fixed-capacity byte buffer with read and write cursors (`nginx ngx_buf_t`): fill from a `net.Conn`, expose unread bytes, consume, compact. Our own tiny `bufio`, written by us. | M1 |
| `buf.Pool` | Hands out and takes back fixed-size buffers via `sync.Pool`; two sizes (8 KiB head buffers, 32 KiB copy buffers). | M1 |
| `http1.Header` | One field line: name and value as received (name case preserved, value trimmed of OWS). | M1 |
| `http1.Headers` | An **ordered** list of `Header` with case-insensitive lookup, `Values`, `Add`, `Set`, `Del`, and `Walk`. Not a map: order and duplicates must survive forwarding (RFC 9110 §5.3). Node's `rawHeaders` rather than its `headers`; Java's `HttpHeaders` loses order. | M1 |
| `http1.RequestHead` | Everything before the body: method, raw target, parsed path and query, version, headers, validated host, body framing, effective `close`, `Expect: 100-continue`, `Upgrade` token. | M1 |
| `http1.ResponseHead` | Version, status, reason, headers, body framing, effective `close`. | M1 |
| `http1.BodyFraming` | The framing decision: `None`, `Length(n)`, `Chunked`, `UntilClose`. Computed once from the headers by the RFC 9112 §6.3 rules. | M1 |
| `http1.ParseError` | A parse failure carrying the status to answer with (400, 431, 501, 505) and a message for the error log. | M1 |
| `http1.ErrNeedMore` | Sentinel: the buffer does not yet hold a complete head; read more and call again. | M1 |
| `http1.BodyReader` | Interface for reading exactly one message body from a `buf.Buffer` plus its `net.Conn`, returning `io.EOF` at the framing boundary and never consuming beyond it. | M1 |
| `http1.LengthReader` | `BodyReader` for `Content-Length`. | M1 |
| `http1.ChunkedReader` | `BodyReader` for `Transfer-Encoding: chunked`; parses chunk sizes, extensions, trailers. | M1 |
| `http1.UntilCloseReader` | `BodyReader` for close-delimited responses. | M1 |
| `http1.ChunkedWriter` | Encodes writes as chunks; `Close` writes the last chunk and the empty trailer. | M1 |
| `http1.Serializer` (functions) | `WriteRequestHead`, `WriteResponseHead`: emit canonical bytes; refuse CR/LF/NUL. | M1 |

```go
// package buf
type Buffer struct {
    b    []byte // len == cap, fixed
    r, w int    // unread bytes are b[r:w]
}
func (b *Buffer) Fill(c io.Reader) (int, error) // read into b[w:], compacting first if needed
func (b *Buffer) Unread() []byte                // b[r:w]
func (b *Buffer) Consume(n int)
func (b *Buffer) Full() bool

// package http1
type Header struct{ Name, Value string }
type Headers []Header

type Version struct{ Major, Minor uint8 }

type RequestHead struct {
    Method    string
    Target    string   // raw request-target as received
    Path      string   // origin-form path, or path of absolute-form
    Query     string
    Version   Version
    Headers   Headers
    Host      string   // validated: exactly one Host, or from absolute-form
    Framing   BodyFraming
    Close     bool     // effective Connection semantics for this request
    Expect100 bool
    Upgrade   string   // M10: "websocket"
}

type ResponseHead struct {
    Version Version
    Status  int
    Reason  string
    Headers Headers
    Framing BodyFraming
    Close   bool
}

type BodyKind uint8
const (BodyNone BodyKind = iota; BodyLength; BodyChunked; BodyUntilClose)
type BodyFraming struct {
    Kind   BodyKind
    Length int64 // valid when Kind == BodyLength
}

var ErrNeedMore = errors.New("http1: incomplete head")
type ParseError struct{ Status int; Msg string }

// Both parsers scan buf for the end of the head, then parse it in one pass.
// n is the number of bytes consumed on success; 0 with ErrNeedMore.
func ParseRequestHead(buf []byte, dst *RequestHead, limits ParseLimits) (n int, err error)
func ParseResponseHead(buf []byte, dst *ResponseHead, limits ParseLimits) (n int, err error)
func ResponseFraming(method string, status int, h Headers) (BodyFraming, error)

type BodyReader interface {
    io.Reader          // io.EOF exactly at the end of this body
    Trailers() Headers // non-nil only for chunked, after EOF
}
func NewBodyReader(f BodyFraming, b *buf.Buffer, src io.Reader) BodyReader

type ChunkedWriter struct{ w io.Writer }
func (cw *ChunkedWriter) Write(p []byte) (int, error) // "%x\r\n" p "\r\n"
func (cw *ChunkedWriter) Close() error                // "0\r\n\r\n"

func WriteRequestHead(dst []byte, h *RequestHead) ([]byte, error)
func WriteResponseHead(dst []byte, h *ResponseHead) ([]byte, error)
```

## 6. Connection and proxy context (package `proxy`)

| Type | Responsibility | Milestone |
|---|---|---|
| `proxy.ClientConn` | Lifecycle of one accepted connection: owns the `net.Conn` and the read buffer, runs the keep-alive loop, applies client-side deadlines, records state for metrics and drain. nginx: `ngx_connection_t` + `ngx_http_connection_t`. | M1 |
| `proxy.ConnState` | `Accepted`, `ReadingHead`, `ReadingBody`, `Proxying`, `WritingResponse`, `Idle`, `Closed`. Drives drain (idle connections are closed at once) and `stub_status` counters. | M1 |
| `proxy.Exchange` | One request/response pair through the proxy: the request head, the matched route, the upstream connection, the response head, byte counters, timings, cache status. nginx: `ngx_http_request_t` + `ngx_http_upstream_t`. | M1 |
| `proxy.Forwarder` (functions) | The header rewrite rules: hop-by-hop removal, `X-Forwarded-*`, `Host`, `Connection`. Pure functions over `Headers`, unit-tested in isolation. | M1 |
| `proxy.ErrorPage` | Locally generated responses (400, 408, 413, 429, 431, 502, 503, 504) with fixed bodies. | M1 |

```go
type ClientConn struct {
    nc      net.Conn            // *net.TCPConn or *tls.Conn
    rb      *buf.Buffer
    lis     *worker.ListenerRT
    remote  netip.AddrPort
    tlsInfo *tlsx.Info          // nil for plaintext
    state   ConnState
    served  int                 // requests on this connection
    w       *worker.Worker
}
func (c *ClientConn) serve() // accept -> loop{ readHead; exchange; keepalive? } -> close

type Exchange struct {
    conn      *ClientConn
    req       http1.RequestHead
    route     *config.Route
    vhost     *config.VirtualHost
    up        *upstream.Conn
    resp      http1.ResponseHead
    started   time.Time
    upstreamT time.Duration
    bytesIn   int64
    bytesOut  int64
    cache     cache.Status       // M8
    tried     []*upstream.Peer   // M7
}
```

## 7. Routing context (package `router`)

| Type | Responsibility | Milestone |
|---|---|---|
| `router.Router` | Per-listener index from host name to `VirtualHost`: exact map, wildcard list, default. Built once per generation. | M3 |
| `router.Match` (function) | Per-virtual-host location search implementing nginx precedence over `Route.Matcher`. | M3 |

```go
type Router struct {
    exact    map[string]*config.VirtualHost
    wildcard []wildcardEntry // "*.example.com" -> suffix ".example.com"
    def      *config.VirtualHost
}
func (r *Router) Host(host string) *config.VirtualHost
func Match(vh *config.VirtualHost, path string) *config.Route
```

## 8. Upstream context (package `upstream`)

| Type | Responsibility | Milestone |
|---|---|---|
| `upstream.Pool` | Runtime for one `config.Upstream`: peers, the balancer, idle connections, the prober. One per worker per upstream. | M1 (trivial), M5, M7 |
| `upstream.Peer` | Runtime for one `config.Backend`: in-flight count, passive failure state (`fails`, `downUntil`), active health flag, smooth-weighted-round-robin counters. | M7 |
| `upstream.Conn` | One TCP (or TLS) connection to a peer with its own read buffer; knows when it went idle and whether it was reused. | M1 |
| `upstream.Dialer` | Dials a peer with `proxy_connect_timeout`, optional TLS (`proxy_ssl_*`). | M1, M5 |
| `upstream.Balancer` | Interface: pick a peer given the request context and the peers already tried. | M7 |
| `upstream.RoundRobin`, `upstream.LeastConn`, `upstream.IPHash` | Balancer implementations; `RoundRobin` uses nginx's smooth weighted algorithm. | M7 |
| `upstream.Prober` | Active health checker: periodic requests to a path, rise/fall thresholds. | M7 |
| `upstream.RetryPolicy` | `proxy_next_upstream` conditions and `proxy_next_upstream_tries`. | M7 |

```go
type Pool struct {
    cfg      *config.Upstream
    peers    []*Peer
    balancer Balancer
    mu       sync.Mutex
    idle     map[*Peer][]*Conn  // M5, bounded by cfg.KeepAlive
    dialer   Dialer
    prober   *Prober            // M7
}
func (p *Pool) Get(ctx PickContext) (*Conn, error) // pick peer, reuse idle or dial
func (p *Pool) Put(c *Conn, reusable bool)

type Peer struct {
    cfg        *config.Backend
    active     atomic.Int32
    fails      atomic.Int32
    downUntil  atomic.Int64 // unix nanos
    healthy    atomic.Bool  // active checks; true when no prober
    curWeight  int          // smooth WRR, guarded by Pool.mu
    effWeight  int
}

type Conn struct {
    nc        net.Conn
    peer      *Peer
    rb        *buf.Buffer
    idleSince time.Time
    reused    bool
}

type Balancer interface {
    Pick(ctx PickContext, peers []*Peer, tried []*Peer) (*Peer, error)
}
```

## 9. TLS context (package `tlsx`)

| Type | Responsibility | Milestone |
|---|---|---|
| `tlsx.ClientHello` | The fields we extract by hand from the raw ClientHello: record version, hello version, SNI, ALPN list, supported versions. | M5 |
| `tlsx.ParseClientHello` (function) | Parses TLS record and handshake framing and the extensions; returns `ErrNeedMore` while records are incomplete. | M5 |
| `tlsx.CertStore` | Maps server names (exact and wildcard) to `*tls.Certificate`, with a default. | M5 |
| `tlsx.prefixConn` | A `net.Conn` that first replays the bytes we peeked, then continues from the real connection, so `crypto/tls` sees an untouched stream. | M5 |
| `tlsx.Info` | Facts about a completed handshake for logging and headers: server name, version, cipher suite. | M5 |

```go
type ClientHello struct {
    RecordVersion     uint16
    HelloVersion      uint16
    ServerName        string
    ALPN              []string
    SupportedVersions []uint16
}
func ParseClientHello(b []byte) (*ClientHello, int, error)

type CertStore struct {
    exact    map[string]*tls.Certificate
    wildcard map[string]*tls.Certificate // key: ".example.com"
    def      *tls.Certificate
}
func (s *CertStore) Lookup(serverName string) *tls.Certificate
```

## 10. Limits, rate limiting, cache, observability, WebSocket, event loop

| Type | Responsibility | Milestone |
|---|---|---|
| `limits.ConnGate` | Counts live connections per worker against `worker_connections`; rejects with 503 when full. | M4 |
| `ratelimit.Zone` | A `limit_req_zone`: rate, key extractor (client IP), bounded table of buckets, sweeper. Per worker. | M6 |
| `ratelimit.Bucket` | nginx's leaky-bucket state: `last` timestamp and `excess` in thousandths of a request. | M6 |
| `ratelimit.Rule` | A `limit_req` on a route: zone, `burst`, `nodelay`/`delay`, reject status. | M6 |
| `ratelimit.Decision` | `Allow`, `Delay(d)`, `Reject`. | M6 |
| `cache.Key` | What identifies a cacheable response: scheme, host, path+query, plus the `Vary` selector; hashed with `sha256` like nginx hashes with md5. | M8 |
| `cache.Policy` | Cacheability decision from method, status, `Cache-Control`, `Expires`, `Vary`, `Set-Cookie`, `Authorization`, and `proxy_cache_valid` overrides. | M8 |
| `cache.Entry` | Stored response: status, headers, body location (memory slice or file), created/expires, validators (`ETag`, `Last-Modified`), size. | M8 |
| `cache.Store` | Interface: `Get`, `Put` (streaming, tee while proxying), `Delete`, `Sweep`. | M8 |
| `cache.MemoryStore` | LRU with a byte budget. | M8 |
| `cache.DiskStore` | Files under a sharded directory (nginx `levels=1:2`), header block + body, index rebuilt on start, LRU eviction by size. | M8 |
| `cache.Status` | `MISS`, `HIT`, `EXPIRED`, `STALE`, `REVALIDATED`, `BYPASS` for `X-Cache-Status` and logs. | M8 |
| `accesslog.Format` | A compiled `log_format`: literal segments and `$variable` segments. | M9 |
| `accesslog.Logger` | Buffered writer to an `O_APPEND` file shared by all workers; reopen on `SIGUSR1`. | M9 |
| `metrics.Counters` | Atomic counters in `stub_status` shape (accepts, handled, requests, active, reading, writing, waiting) plus per-status and per-upstream counts; exported as text. | M9 |
| `proxy.Tunnel` | After a `101 Switching Protocols`: flush leftover bytes both ways, then copy bytes bidirectionally with half-close propagation and idle timeout. | M10 |
| `epoll.Loop` | Milestone 11 experiment: `epoll_create1`, non-blocking `accept4`, `EPOLLIN|EPOLLOUT|EPOLLET`, a timer heap, and `ConnFSM` per connection reusing the `http1` parser. | M11 |

```go
// package ratelimit (nginx ngx_http_limit_req_module arithmetic)
type Bucket struct {
    lastMs int64 // milliseconds
    excess int64 // requests * 1000
}
type Zone struct {
    name    string
    rateMs  int64 // requests per second * 1000
    mu      sync.Mutex
    buckets map[string]*Bucket
    max     int
}
func (z *Zone) Take(key string, rule *Rule, now time.Time) Decision

// package cache
type Store interface {
    Get(k Key) (*Entry, Status)
    Put(k Key, e *Entry) (io.WriteCloser, error) // body is streamed in while proxying
    Delete(k Key)
}
```

## 11. Type index

| Package | Type | Milestone |
|---|---|---|
| `master` | `Master`, `WorkerHandle`, `Generation`, `WorkerState` | M2 |
| `worker` | `Worker`, `ListenerRT` | M1, M2 |
| `proc` | `ControlConn`, `Inherited` helpers | M2 |
| `config` | `Directive`, `Config`, `Listener`, `VirtualHost`, `Route`, `Matcher`, `Action`, `ProxyAction`, `ReturnAction`, `Upstream`, `Backend`, `Limits`, `TLSSettings`, `UpstreamTLS`, `RateZone`, `RateRule`, `CachePath`, `CacheRule`, `AccessLogSpec`, `ActiveCheck`, `RetryPolicy` | M3 onward |
| `buf` | `Buffer`, `Pool` | M1 |
| `http1` | `Header`, `Headers`, `Version`, `RequestHead`, `ResponseHead`, `BodyKind`, `BodyFraming`, `ParseError`, `ErrNeedMore`, `ParseLimits`, `BodyReader`, `LengthReader`, `ChunkedReader`, `UntilCloseReader`, `ChunkedWriter` | M1 |
| `proxy` | `ClientConn`, `ConnState`, `Exchange`, forwarding functions, `ErrorPage`, `Tunnel` | M1, M10 |
| `router` | `Router`, `Match` | M3 |
| `upstream` | `Pool`, `Peer`, `Conn`, `Dialer`, `Balancer`, `RoundRobin`, `LeastConn`, `IPHash`, `Prober`, `RetryPolicy` | M1, M5, M7 |
| `tlsx` | `ClientHello`, `CertStore`, `prefixConn`, `Info` | M5 |
| `limits` | `ConnGate` | M4 |
| `ratelimit` | `Zone`, `Bucket`, `Rule`, `Decision` | M6 |
| `cache` | `Key`, `Policy`, `Entry`, `Store`, `MemoryStore`, `DiskStore`, `Status` | M8 |
| `accesslog` | `Format`, `Logger` | M9 |
| `metrics` | `Counters` | M9 |
| `epoll` | `Loop`, `ConnFSM` | M11 |

## 12. Invariants worth writing down

1. A `RequestHead` is valid only after `ParseRequestHead` returned `n > 0`; callers
   never inspect a partially filled head.
2. A `BodyReader` returns `io.EOF` exactly at the framing boundary and leaves any
   further bytes in the `buf.Buffer` for the next request.
3. A `ClientConn` moves to `Idle` only if the previous request body was fully read and
   the response was fully written; otherwise it closes.
4. An `upstream.Conn` goes back to the pool only if the response body was fully read
   and neither side sent `Connection: close`.
5. The serializer never emits a byte sequence containing CR or LF inside a name or
   value. The parser rejects them first; the serializer is the second line.
6. Config values are never mutated after `config.Load` returns. Runtime state lives
   only in `worker`, `upstream`, `ratelimit`, `cache`, `accesslog`, `metrics`.
7. All `[]byte` slices returned by `buf.Buffer` are invalidated by the next `Fill` or
   `Consume`; copy out if you need to keep them.

## 13. Comparisons for orientation

| Concept | Java / Node analogue | Difference here |
|---|---|---|
| `Headers` ordered list | Node `rawHeaders` array; Java `HttpHeaders` map | We keep order and duplicates because we are a proxy, not an app |
| `buf.Buffer` | Netty `ByteBuf` with `readerIndex`/`writerIndex`; Node `Buffer` + offsets | Fixed capacity, pooled, no growth |
| `ClientConn.serve` loop | Servlet container connection thread; Node `net.Socket` event handlers | One goroutine, sequential code, runtime parks it on I/O |
| `Exchange` | `HttpServletRequest` + `HttpServletResponse` + the proxy client call | Streaming both ways; no body object |
| `upstream.Pool` | Apache HttpClient `PoolingHttpClientConnectionManager`; Node `http.Agent` | Per worker, bounded by `keepalive N` |
| `config.Directive` AST | Synapse XML DOM before it is turned into mediators | Same two-stage idea: syntax tree, then typed schema |
