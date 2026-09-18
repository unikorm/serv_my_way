# mini-nginx: Repository Layout and Testing Strategy

> Status: Phase 1 design draft. The tree below is the target shape; packages appear
> when their milestone starts.

## 1. Package tree

```
serv_my_way/
├── go.mod                         module path: see OPEN_DECISIONS #12; go 1.27
├── Makefile                       build, test, vm-test, bench targets (no dependencies)
├── README.md
├── docs/                          these documents, plus per-milestone notes as we go
├── cmd/
│   └── mini-nginx/
│       ├── main.go                //go:build linux  : dispatch "master" (default), "worker", -t, -s
│       └── main_other.go          //go:build !linux : prints "mini-nginx runs on Linux only"
├── internal/
│   ├── buf/                       [M1]  Buffer, Pool                       (portable)
│   ├── http1/                     [M1]  parser, serializer, body readers   (portable)
│   │   └── testdata/              raw request/response fixtures
│   ├── proxy/                     [M1]  ClientConn, Exchange, forwarding rules, error pages, Tunnel [M10]
│   ├── upstream/                  [M1]  Pool, Peer, Conn, Dialer; Balancer [M7]; Prober [M7b]
│   ├── worker/                    [M1]  Worker, ListenerRT, accept loop, drain [M2]
│   ├── master/                    [M2]  Master, WorkerHandle, signals, reload      (linux)
│   ├── proc/                      [M2]  fd inheritance, control socket, Pdeathsig, SO_REUSEPORT (linux)
│   ├── config/                    [M3]  lexer, parser (Directive AST), schema (Config)
│   │   └── testdata/              .conf fixtures, valid and invalid
│   ├── router/                    [M3]  host and location matching
│   ├── limits/                    [M4]  ConnGate, deadline helpers
│   ├── tlsx/                      [M5]  ClientHello parser, CertStore, prefixConn
│   │   └── testdata/              captured ClientHello hex dumps
│   ├── ratelimit/                 [M6]
│   ├── cache/                     [M8]
│   ├── accesslog/                 [M9]
│   ├── metrics/                   [M9]
│   └── epoll/                     [M11] event loop                          (linux)
├── test/
│   ├── backend/                   Go test backend (net/http allowed here: it is not the proxy)
│   │   └── main.go                /echo, /slow, /nolength, /hang, /whoami, /healthz, /ws
│   ├── integration/               //go:build integration && linux
│   │   ├── main_test.go           builds the binary, starts backend(s), starts mini-nginx
│   │   ├── proxy_test.go          M1 behaviours over raw net.Dial
│   │   ├── smuggling_test.go      raw byte attacks, expects 400 + close
│   │   ├── reload_test.go         M2: load + SIGHUP, zero errors
│   │   └── ...
│   └── conf/                      configs used by integration tests and manual runs
├── scripts/
│   ├── vm-test.sh                 run unit + integration tests inside the OrbStack VM
│   ├── vm-run.sh                  cross-compile and run the binary in the VM with a config
│   ├── bench.sh                   wrk / hey runs for M11 comparisons
│   └── gen-certs.sh               self-signed CA + per-host certs for M5 tests
├── testdata/certs/                generated, git-ignored
└── .gitattributes                 keeps byte fixtures untouched by line-ending conversion
```

Why `internal/`: Go forbids importing `internal/...` from outside the module, which
keeps every package private without ceremony. Java analogy: package-private at module
scope. Nothing in this project is a library for others.

Package naming follows nginx's vocabulary where a counterpart exists: `http1` is
`ngx_http_parse.c`, `proxy` is `ngx_http_proxy_module`, `upstream` is
`ngx_http_upstream_*`, `master` is `ngx_process_cycle.c`.

## 2. Build tags and cross-compiling from macOS

Two groups of code:

- **Portable**: `buf`, `http1`, `proxy`, `router`, `config`, `upstream`, `tlsx`,
  `ratelimit`, `cache`, `accesslog`, `metrics`. They use only `net`, `crypto/tls`,
  `time`, `sync`, and friends, so their unit tests run natively on the Mac.
- **Linux-only**: `master`, `proc`, `epoll`, `cmd/mini-nginx`. Every file carries
  `//go:build linux` and the package has a `doc.go` so it still exists on other
  platforms.

Commands:

```sh
go test ./...                                          # on the Mac: portable packages only
GOOS=linux GOARCH=arm64 go vet ./...                   # type-check the Linux code without leaving the Mac
GOOS=linux GOARCH=arm64 go build -o bin/mini-nginx ./cmd/mini-nginx   # static binary, no cgo, no deps
```

Cross-compiling is trivial precisely because there are no dependencies and no cgo.

## 3. OrbStack workflow

One Linux machine for the whole project:

```sh
orb create ubuntu:24.04 mininginx
orb -m mininginx sudo apt-get install -y curl netcat-openbsd wrk hey websocat moreutils
# Go inside the VM (needed for `go test -tags integration`; the binary itself can be cross-compiled)
orb -m mininginx bash -c 'curl -fsSL https://go.dev/dl/go1.27.1.linux-arm64.tar.gz | sudo tar -C /usr/local -xz'
```

OrbStack exposes the Mac filesystem inside the machine and maps the current working
directory when you run `orb <command>` from a project folder; verify once with
`orb -m mininginx pwd`. If Go compilation over the shared filesystem feels slow, keep a
clone in the VM's own home directory and `git pull` there; the scripts take a `-C dir`
argument for that.

`scripts/vm-test.sh` does, in the VM: `go vet ./... && go test -race ./... && go test
-race -tags integration ./test/integration/...`. `scripts/vm-run.sh` cross-compiles on
the Mac, then runs `./bin/mini-nginx -c test/conf/dev.conf` in the VM in the foreground
so you can send signals from another terminal.

Ports: the VM's ports are reachable from the Mac at the machine's hostname
(`mininginx.orb.local`), so `curl` from the Mac works too.

## 4. Testing strategy

```
                 +---------------------------+
                 | manual: curl, nc, wrk,    |   exploration, definition-of-done checks
                 | openssl s_client, ss      |
                 +---------------------------+
              +---------------------------------+
              | integration (VM, build tag)     |   real sockets, real backend, real signals
              | binary under test as a process  |
              +---------------------------------+
        +-----------------------------------------------+
        | unit: table-driven, byte fixtures, fuzzing,   |   fast, run on the Mac, most of the tests
        | split-at-every-byte, race detector            |
        +-----------------------------------------------+
```

### 4.1 Unit tests with raw byte fixtures

- **Small cases live as Go string literals** in table-driven tests, because `\r\n`
  is visible and cannot be mangled by editors or git:

  ```go
  {name: "space before colon", in: "GET / HTTP/1.1\r\nHost: x\r\nFoo : bar\r\n\r\n", wantStatus: 400},
  ```

- **Large captures live in `testdata/`** as `.http` files (raw bytes) or `.hex` files
  (hex dump, one record per line). Hex is preferred for anything binary (TLS) since it
  is immune to line-ending conversion. `.gitattributes` contains
  `*.http -text` and `testdata/** -text` so git never touches CRLF.
- **Split-at-every-byte test.** For each valid fixture of length `L`, feed the parser
  `in[:k]` for every `k < L` and require `ErrNeedMore`, then feed `in` and require the
  same result as parsing it whole. This catches every "assumed the bytes are all
  there" bug and is the test that keeps the milestone 11 event loop possible.
- **Round-trip tests** for `ChunkedWriter` into `ChunkedReader`, with random chunk
  boundaries.
- **Fuzzing** with the standard `testing.F`: `FuzzParseRequestHead`,
  `FuzzParseResponseHead`, `FuzzChunkedReader`, `FuzzParseClientHello`,
  `FuzzConfigParser`. Seed corpus from the fixtures. The invariant under fuzzing is
  "never panic, never read out of bounds, never consume more than returned".
- **Race detector** always on: `go test -race`.
- **Pure functions** for the forwarding rules (hop-by-hop, `X-Forwarded-*`) and the
  framing decision, tested exhaustively as tables.
- **Golden files** for the serializer and the access-log format.

### 4.2 Integration tests in the VM

- Build tag `integration` and Linux only, so `go test ./...` on the Mac stays fast.
- `TestMain` builds `cmd/mini-nginx` into a temp dir, starts the Go test backend on a
  random port, writes a config, and starts mini-nginx as a child process. Tests talk
  to it with `net.Dial` and hand-written bytes, never with `net/http` on the client
  side either, so the tests exercise the same low-level path.
- The backend (`test/backend`) may use `net/http`: it stands in for your real
  applications and is explicitly outside the "no high-level HTTP" constraint. It
  echoes received headers as JSON, and offers `/slow`, `/nolength`, `/hang`,
  `/whoami`, `/healthz`, and a WebSocket echo on `/ws`. The backend is not bound by
  the project's constraints, so its `/ws` echo can be a minimal frame parser in Go or
  simply a Node `ws` echo server started by the test.
- Process-model tests send real signals to the child and assert on pids and on `wrk`
  or in-test load results.
- TLS tests generate a CA and per-host certificates with `crypto/x509` in `TestMain`
  (`scripts/gen-certs.sh` does the same for manual runs with `openssl`).

### 4.3 Benchmarks (milestone 11, and spot checks earlier)

- `go test -bench` micro-benchmarks for the parsers with `b.ReportAllocs()` so
  allocation counts are tracked from M1.
- `scripts/bench.sh`: same backend, same VM, `wrk -t4 -c{100,1000,5000} -d30s`, run
  against mini-nginx (goroutine mode), mini-nginx (epoll mode), and nginx installed
  from apt with an equivalent config. `runtime/pprof` CPU and heap profiles written to
  files (`net/http/pprof` is off limits; the file-based API is enough).

## 5. Tooling and conventions

- `gofmt` and `go vet` are mandatory; both ship with Go. `staticcheck` is optional
  developer tooling, not a module dependency, so it does not violate the constraint;
  your call.
- Errors: wrap with `fmt.Errorf("...: %w", err)`; sentinel errors for control flow
  (`ErrNeedMore`); typed `ParseError` for anything that maps to a status code.
- Logging: `log/slog` for the error log, our own `accesslog` for access lines.
- Every milestone ends with a short `docs/notes/Mn.md`: what was built, what was
  simplified, hexdumps captured along the way.
- `Makefile` targets: `build`, `test`, `vet`, `fuzz`, `vm-test`, `vm-run`, `bench`.
