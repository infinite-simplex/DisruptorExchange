# DisruptorExchange

[![CI](https://github.com/infinite-simplex/DisruptorExchange/actions/workflows/ci.yml/badge.svg)](https://github.com/infinite-simplex/DisruptorExchange/actions/workflows/ci.yml)

A low-latency exchange pipeline built around an LMAX Disruptor-style
core — lock-free SPSC ring buffers, a replicated and leader-elected matching
tier with real failover, a sequenced ingress gateway with a memory-mapped log
and gap-fill retransmit. The pipeline is product-agnostic: `NetworkGateway` and
`MatchingService` are templated on the wire protocol and command type, so a
product plugs its own matching logic into the same ingress → replication →
egress pipeline rather than rebuilding it.

This is a portfolio project, not a production exchange.

## What's here

Just the reusable core (`Core/`) and its telemetry/aggregation tooling
(`PerformanceService/`) — no product logic. Products consume this repo as a
dependency rather than living inside it.

## Products built on it

- **[EventContractLOB](https://github.com/infinite-simplex/EventContractLOB)**
  — a limit order book for binary-outcome event contracts that speaks NASDAQ
  OUCH 4.2 on ingress. The reference implementation; start there to see the
  pipeline in use.
- **[BatchGPU](https://github.com/infinite-simplex/BatchGPU)** — an hourly
  batch auction for GPU compute capacity, proving the pipeline generalizes
  beyond a continuous limit order book.

Design rationale and tradeoffs are written up in the
[DisruptorExchange Core design write-up](https://infinite-simplex.github.io/InfiniteLabs/#DisruptorExchangeCore).

## What a product must provide

Every replica applies the same sequenced command stream to its own engine,
so a product's engine must be **deterministic**: the same commands, applied
in order from sequence 0, give the same state and the same events on every
replica. Core relies on this for replay (late join and restart) and for
failover without any hand-over of state. In practice:

- Nothing that affects state depends on hash-container order, pointer values
  or randomness.
- Nothing that affects state depends on a floating-point result that can
  differ between hosts.
- A clock reading may stamp an emitted event. It must never feed back into
  state.
- The command stream is the only input. Anything that must affect every
  replica (a timed cancel, a batch clear) enters through the gateway as a
  sequenced command and does not fire locally.
- A command that makes the engine throw must throw on every replica.

## Building and running

Linux on x86-64 only (uses `epoll`, `mmap`, multicast sockets and a calibrated
TSC clock that falls back to `CLOCK_TAI` — on Windows, build under WSL). This repo alone only builds `Core` and
`PerformanceService`; there's no standalone executable here — see a product
repo (e.g. EventContractLOB) to run the exchange end to end.

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j"$(nproc)"
```

`-DDISRUPTOREXCHANGE_BENCHMARKS=ON` also builds the opt-in benchmark tools in
`Benchmarks/` (`IngressBench` runs the real gateway and receivers without a
product; the write-up's ingress and Aeron comparison numbers came from it).

## Testing

```bash
ctest --test-dir build --output-on-failure
```

Core primitives only: replica arbitration, telemetry, performance
aggregation, the ingress recovery path (sequence store, retransmit server,
multicast receiver — late join, restart, duplicates, lost tails, damaged
retransmit replies), the ingress gateway with a stand-in wire protocol
(sequencing, store-before-multicast, datagram batching, multi-datagram sends,
split and broken frames, a client whose socket is full), the calibrated TSC clock
(scripted slews, steps and a preempted calibrator, plus the real clock), and egress (batched leader-only
publishing, epoch fencing, ordinal accounting and loss reports, failover to the
surviving replica). The tests use stand-in command and event types. A
product repo carries its own tests for whatever it builds on top (matching
engine, real wire protocol). CI runs this on every push.

## Known limits

Core's own limits; a product lists its own in its repo. The
[write-up](https://infinite-simplex.github.io/InfiniteLabs/#DisruptorExchangeCore)
has the detail.

- **One gateway.** The sequencer, sequence store, retransmit server, arbiter and
  egress gateway live in one process, which is not fault tolerant.
- **The log is a bounded buffer, not storage.** The sequence store is a
  memory-mapped ring (2^20 slots by default) that is zeroed every time the gateway
  starts. It lets live replicas recover lost datagrams; it is not crash-recovery
  storage. A replica that starts or restarts replays from sequence 0, which works only
  while the newest sequence number is below the ring's capacity, and replicas must be
  restarted along with the gateway.
- **Election is a central arbiter with a fencing token**, not a consensus protocol
  among the replicas, and it re-evaluates only when a heartbeat arrives.
- **Egress is one thread** that receives, encodes and delivers every event. Tuned, it
  used about a third of a core at 500,000 orders/s and lost nothing, but more flow
  can overwhelm it.
- **Idle cost.** Three threads per replica spin at 100% whether or not traffic
  arrives, and nothing is pinned or tuned.
- **The trace arena has 2^20 slots.** A command that rests long enough for the ring
  to wrap loses its timing record; a counter shows it.
- **Every measurement is from one machine** with every process on it and the
  multicast groups on loopback, so none of them measures a network.

## License

[MIT](LICENSE).
