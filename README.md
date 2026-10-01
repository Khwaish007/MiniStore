# ministore

A small distributed key-value database with the same shape as a cloud database service: a **C++ storage engine** (data plane), a **Go control plane** (cluster membership, shard routing, billing), and a **terminal dashboard** instead of a web UI.

> Built as a portfolio project targeting SingleStore's PS-II project ("Features in Database Engineering, Cloud Native Services" — C++, Go, Python, React; database core engine + cloud billing / control-plane / data-plane services).

## Architecture

```
        minictl (Go, terminal UI)
              |  gRPC (control.proto)
              v
     control plane (Go)
       - consistent-hash shard ring
       - node registry / heartbeats
       - per-tenant billing meter
              |  gRPC (storage.proto), fan-out per shard
              v
   storage node A       storage node B       storage node C
   (C++ engine)         (C++ engine)         (C++ engine)
```

| Component | Description |
|---|---|
| `storage-engine/` (C++) | The **data plane**. A single-node LSM-lite KV engine: memtable + write-ahead log (real `fsync` durability) + immutable sorted SSTable files with a sparse index and a Bloom filter, background size-tiered compaction, and a merge-iterator-backed range `Scan` — all exposed over gRPC. This is the "database core engine" piece. |
| `control-plane/` (Go) | The **control plane**. Owns cluster membership (`RegisterNode` / `Heartbeat`, with a reaper that evicts unhealthy nodes from the ring and a heartbeat-driven self-heal back in), a consistent-hashing ring that maps each tenant to an owning storage node, and a billing meter that turns routed read/write ops + stored bytes into an estimated cost per tenant. |
| `control-plane/cmd/minictl` | The **terminal dashboard**: polls the control plane and renders node health (including eviction state) and per-tenant billing as refreshing text tables. No web frontend. |
| `proto/` | The two gRPC contracts: `storage.proto` for the data plane, `control.proto` for the control plane / client / admin API. |
| `storage-engine/Dockerfile`, `control-plane/Dockerfile`, `docker-compose.yml` | A 3-storage-node + 1-control-plane cluster you can bring up with one command (see [Running it](#running-it)). |

Keys are namespaced per tenant at the storage node's RPC boundary (`tenant_id + '\0' + key`), so the engine itself stays tenant-agnostic and all multi-tenancy/billing logic lives in the control plane, which sees every request as it routes it.

## Status

- [x] **Storage engine core:** memtable, WAL (append + replay, real `fsync` durability), SSTable writer/reader with sparse index + Bloom filter, background size-tiered compaction with tombstone GC, range `Scan` via a shared merge iterator, crash-recovery-safe `Engine` (`storage-engine/src/`). Unit-tested, including a ThreadSanitizer pass on every concurrency-touching test.
- [x] **Control plane:** consistent-hash shard ring, node registry with a reaper (evicts unhealthy nodes from the ring, self-heals on a fresh heartbeat), billing meter (`control-plane/internal/`). Unit-tested.
- [x] **gRPC contracts (`proto/`)**, both servers wired up and verified live: storage nodes register with the control plane on startup (retry + backoff, standalone-if-unreachable) and heartbeat every 5s; the control plane routes client Put/Get/Delete to the right node and meters usage.
- [x] **Dockerfiles + docker-compose** for a 3-node cluster, and **CI** (`.github/workflows/ci.yml`: C++ build + test, Go build + test + vet + race, both Docker images).
- [x] **Load-testing tool** (`control-plane/cmd/loadgen`, open-loop / Poisson) and a standalone compaction / Bloom-filter benchmark (`storage-engine/tools/bench_compaction.cc`). Results and the real tradeoffs behind every design decision above are written up in [`docs/DESIGN.md`](docs/DESIGN.md) — including a benchmark-confirmed finding that the control plane's per-request gRPC dial (no connection pooling), not storage-node count, is the actual bottleneck under load.

### Explicitly out of scope for this pass (backlog, not attempted)

- Replication / HA
- Per-tenant auth / rate limiting
- Prometheus metrics
- Kubernetes manifests
- Group-commit WAL batching
- gRPC connection pooling in the control plane
- TLS
- A SQL-lite query layer

## Setup

The storage engine's core library, CLI, and tests build today with just `cmake` / `g++` (already verified). The gRPC servers need protobuf/gRPC (C++) and Go + the protoc Go plugins:

```bash
sudo apt install golang-go protobuf-compiler libprotobuf-dev libgrpc++-dev protobuf-compiler-grpc

go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
export PATH="$PATH:$(go env GOPATH)/bin"   # so protoc can find the plugins above

cd control-plane && go mod tidy            # fetches google.golang.org/grpc, /protobuf
```

Then, from the repo root:

```bash
make proto        # generates control-plane/gen/{controlpb,storagepb}
make cpp-build    # re-run; CMake now finds gRPC and adds the storage_server target
make cpp-test
make go-build
make go-test
```

## Running it

### Docker Compose (3-node cluster, easiest)

```bash
docker compose up --build   # 1 control plane + 3 storage nodes
```

The control plane's port is published to the host, so the dashboard runs directly from the host against it:

```bash
cd control-plane && go run ./cmd/minictl --control localhost:8080
```

You should see all three nodes register themselves (by their compose service DNS names) and go healthy within a few seconds.

### Manually (single node)

```bash
make run-storage   # storage node on 0.0.0.0:9090, data in ./data/node1,
                   # registers with the control plane at localhost:8080
make run-control   # control plane on :8080
make run-cli       # terminal dashboard, polls :8080 every 2s
```

## Benchmarking

```bash
make bench   # runs bench_compaction, then loadgen against localhost:8080
             # (bring up a cluster first, per "Running it" above)
```

- `storage-engine/build/bench_compaction` measures compaction's and the Bloom filter's effect directly against the SSTable layer, no gRPC needed.
- `control-plane/cmd/loadgen` is an open-loop (Poisson arrival) load generator against a running control plane — see its `--help` for workers / rate / duration / tenants / key-space flags and an optional `--csv` output path.

Results and the design decisions behind them are written up in [`docs/DESIGN.md`](docs/DESIGN.md).

## Debugging the engine directly

`storage_cli` is a REPL over the raw `Engine`, no gRPC required — useful for poking at the storage layer in isolation:

```bash
./storage-engine/build/storage_cli ./data/debug
```

```
> put foo bar
OK
> get foo
bar
> stats
keys(memtable)=1 bytes(memtable)=6 reads=1 writes=1
```
