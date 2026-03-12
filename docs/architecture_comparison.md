# Architecture Comparison — Industry Research vs PyEventBT

> Cross-reference of industry patterns (LMAX, NautilusTrader, LEAN) against PyEventBT's current implementation. See also: [Industry Research](industry_research.md) | [Design Pattern](design_pattern.md) | [Distributed Migration](distributed_migration.md) | [Contracts & Protocols](contracts_protocols.md)

---

## 32.1 Summary Verdict

PyEventBT's **core event loop pattern is industry-validated** — LMAX, NautilusTrader, and LEAN all use a single-threaded sequential event loop as their architectural foundation. However, PyEventBT diverges from industry best practices in **state management**, **inter-component communication**, **broker abstraction**, and **resilience patterns**. These divergences are the specific areas to address when building a new system.

| Area | Industry Pattern | PyEventBT | Alignment |
|---|---|---|---|
| Core event loop | Single-threaded, sequential dispatch | `TradingDirector` single-threaded loop | **Aligned** |
| Event types | Typed, immutable event objects | Dataclass events (BAR, SIGNAL, ORDER, FILL) | **Aligned** |
| Backtest = Live | Same pipeline, swap connectors | Same pipeline, swap DataProvider + ExecutionEngine | **Aligned** |
| State management | Cache / single-writer / event sourcing | `SharedData` mutable singleton | **Divergent** |
| Inter-component comm | MessageBus (pub/sub + req/resp) | `queue.Queue` (simple FIFO) | **Partial** |
| Broker abstraction | Ports & Adapters / formal interfaces | Connectors (informal, 2 implementations) | **Partial** |
| Process separation | I/O separated, logic stays together | Everything in one process | **Aligned** (by default) |
| Crash recovery | Event sourcing + replay | None | **Gap** |
| Pre/post-trade risk | Separate concerns, different processes | Single `RiskEngine`, no post-trade | **Gap** |
| Multi-venue | Native adapter-per-venue | Single broker (MT5) | **Gap** |

---

## 32.2 Detailed Comparison by Pattern

### 32.2.1 Event Loop — Aligned

| | Industry (LMAX/Nautilus/LEAN) | PyEventBT |
|---|---|---|
| **Threading** | Single-threaded BLP / Kernel / AlgorithmManager | Single-threaded `TradingDirector` |
| **Dispatch** | Dequeue → route by type → handler | `queue.get()` → `handler_map[event.type](event)` |
| **Ordering** | Sequential, fully process before next | Sequential, fully process before next |
| **Determinism** | Guaranteed by single-threaded design | Guaranteed by single-threaded design |

**Assessment:** This is the strongest alignment point. PyEventBT's `TradingDirector` is architecturally equivalent to LMAX's BLP, NautilusTrader's `NautilusKernel`, and LEAN's `AlgorithmManager`. All four process events one at a time on a single thread. This design is validated at production scale (LMAX processes 6M orders/sec on one thread).

**For your system:** Keep this pattern. It is the industry-proven foundation.

---

### 32.2.2 State Management — Divergent

| | Industry | PyEventBT |
|---|---|---|
| **Pattern** | Single Writer Principle | Multiple writers to `SharedData` |
| **Storage** | Dedicated Cache object (NautilusTrader) or in-BLP memory (LMAX) | Class-level static attributes on `SharedData` |
| **Access** | Kernel/BLP owns writes; components read | ExecutionEngine + connectors both write; everyone reads |
| **Recovery** | Event sourcing — replay log to rebuild state | None — state lost on crash |
| **Thread safety** | Single writer = no locks needed | `__dict__` mutation, no thread safety |

**Assessment:** This is the largest divergence. Every industry architecture enforces the **Single Writer Principle**: exactly one component owns state mutations, and all other components read from that authoritative source.

PyEventBT's `SharedData`:
- Uses class-level (static) attributes — essentially a global mutable singleton
- Is written by both `ExecutionEngine` and MT5 connectors (violates single writer)
- Uses `__dict__` mutation (`SharedData.account_info.__dict__["balance"] = ...`) — fragile and untraceable
- Has no change notification — readers poll stale values

**What NautilusTrader does instead:**
```
Kernel (single writer) → Cache (in-memory store) → Components (read-only access)
                       ↘ Event log (journal) → Recovery via replay
```

**For your system:**
1. Replace `SharedData` with a `Cache` object owned by the event loop
2. Only the loop/kernel writes to the Cache after processing events
3. Components receive read-only snapshots or query the Cache through a defined interface
4. Optionally journal all state-changing events for replay/recovery

---

### 32.2.3 Inter-Component Communication — Partially Aligned

| | Industry | PyEventBT |
|---|---|---|
| **Mechanism** | MessageBus with multiple patterns | `queue.Queue` (FIFO only) |
| **Patterns supported** | Pub/sub, request/response, command/event | Put/get only |
| **Multiple consumers** | Yes — multiple subscribers per topic | No — one consumer dequeues |
| **Request/response** | Native (NautilusTrader) | Not available — uses sync calls via `Modules` |
| **Topic routing** | By event type, symbol, strategy | By event type only (in dispatch map) |

**Assessment:** PyEventBT's `queue.Queue` is the right starting point (LMAX also uses a simple ring buffer), but it lacks the communication patterns that more mature systems use:

- **Pub/sub**: NautilusTrader's MessageBus lets multiple components subscribe to the same event type. PyEventBT's dispatch map allows only one handler per event type.
- **Request/response**: When a SignalEngine needs current bars or positions, NautilusTrader uses the MessageBus's request/response pattern. PyEventBT bypasses the queue entirely via `Modules` direct object references.
- **Commands vs Events**: Industry systems distinguish between commands (do something) and events (something happened). PyEventBT mixes both — `OrderEvent` is really a command, while `FillEvent` is a true event.

**The `Modules` bypass problem:** PyEventBT gives user callbacks direct object references through `Modules`, letting them call `DATA_PROVIDER.get_latest_bars()` and `EXECUTION_ENGINE.close_positions()` synchronously. This works in single-process but:
- Violates the "no direct component-to-component calls" invariant stated in the design pattern
- Makes distribution impossible without replacing `Modules` with RPC proxies
- Creates hidden dependencies not visible in the event flow

**For your system:**
1. Start with a simple queue (like PyEventBT), but design it as a `MessageBus` interface from day one
2. Support pub/sub (multiple handlers per event type)
3. Add request/response for synchronous queries (bars, positions) — route through the bus, not direct calls
4. Separate commands ("place this order") from events ("this order was filled") in your type system

---

### 32.2.4 Broker Abstraction — Partially Aligned

| | Industry | PyEventBT |
|---|---|---|
| **Pattern** | Ports & Adapters (hexagonal) | Connectors (informal) |
| **Interface formality** | Strict interfaces (IBrokerage in LEAN, Adapter in Nautilus) | `IExecutionEngine` exists but connectors don't fully implement it |
| **Multi-venue** | Native — one adapter per venue, RoutingEngine dispatches | Single broker (MT5), no routing |
| **Discovery** | Adapters register capabilities | Hardcoded in configuration |

**Assessment:** PyEventBT has the right instinct — it uses connectors to abstract the broker layer, and `IExecutionEngine` + `IDataProvider` interfaces exist. But the implementation is informal:

- Only 2 connectors exist (MT5 simulator + MT5 live), both tightly coupled to MT5 specifics
- The connector pattern is not documented as a formal adapter interface with required methods
- No routing layer exists for multi-venue scenarios

**NautilusTrader's approach:**
```
Strategy → ExecutionEngine → RoutingEngine → Adapter (per venue)
                                            ├── BinanceAdapter
                                            ├── InteractiveBrokersAdapter
                                            └── SimulatedExchangeAdapter
```

Each adapter implements a standard interface: `submit_order()`, `cancel_order()`, `modify_order()`, `connect()`, `disconnect()`. The routing engine maps `venue_id` to the correct adapter.

**For your system:**
1. Define a formal `IAdapter` interface with required methods (connect, disconnect, submit_order, cancel_order, subscribe_data, etc.)
2. Build a `RoutingEngine` that maps venue/broker to adapter instances
3. Each broker gets its own adapter implementation — MT5, IBKR, Binance, etc.
4. Adapters handle serialization, connection management, and protocol translation (e.g., FIX)
5. The rest of the system never knows which broker is being used

---

### 32.2.5 Process Separation — Aligned (Current State is Correct)

| | Industry Consensus | PyEventBT |
|---|---|---|
| **Core logic** | Keep in one process | In one process |
| **Market data I/O** | Separate thread/process | In same process |
| **Broker I/O** | Separate thread/process | In same process (live uses MT5 thread) |
| **Persistence** | Separate thread/process | In same process |
| **Monitoring** | Separate service | None |

**Assessment:** PyEventBT's single-process architecture is actually **correct for backtesting** — all three reference architectures (LMAX, LEAN, NautilusTrader) run as single processes for their core loop. The industry only separates I/O-bound components.

The validated process split for live trading is:

| Process | Contains | Why separate |
|---|---|---|
| **A: Market Data** | Data adapters, normalization, fan-out | I/O bound (network), can restart independently |
| **B: Strategy Core** | SignalEngine + SizingEngine + RiskEngine + Cache + MessageBus | Computation — must be low-latency, single-threaded |
| **C: Execution** | Broker adapters, order routing | I/O bound (broker API), different reliability needs |
| **D: Monitoring** | Portfolio view, trade archive, risk dashboard | Read-only, can tolerate latency |

**For your system:**
1. Keep backtest as single-process (no benefit to distributing during replay)
2. For live: separate data ingestion and execution into their own processes
3. Use ZeroMQ or Aeron between data→strategy and strategy→execution (low latency)
4. Use Kafka between strategy→monitoring (persistence, replay, multiple consumers)
5. Keep Signal+Sizing+Risk in the same process — they need microsecond access to shared state

---

### 32.2.6 Crash Recovery & Event Sourcing — Gap

| | Industry | PyEventBT |
|---|---|---|
| **Recovery mechanism** | Event sourcing — replay journal from last snapshot | None |
| **State persistence** | Event log + periodic snapshots | In-memory only |
| **Deterministic replay** | Native (replay same events = same state) | Possible in theory, not implemented |
| **Audit trail** | Every state change is a persisted event | None |

**Assessment:** Both LMAX and NautilusTrader use event sourcing as a core architectural feature. LMAX calls it out explicitly: "current state is entirely derivable by processing the input events." NautilusTrader uses "crash-only design" with externalized state.

PyEventBT has no recovery mechanism. If the process crashes during live trading, all in-memory state (open positions tracking, account balance, pending orders) is lost.

**For your system:**
1. Journal every input event (BarEvent, FillEvent from broker) to an append-only log
2. Take periodic snapshots of the Cache state
3. On startup: load last snapshot, replay events since snapshot
4. This also gives you deterministic backtest replay and a complete audit trail

---

### 32.2.7 Pre-Trade vs Post-Trade Risk — Gap

| | Industry | PyEventBT |
|---|---|---|
| **Pre-trade risk** | Synchronous, in-process, before order submission | `RiskEngine` in pipeline |
| **Post-trade risk** | Separate service, consumes fills asynchronously | Not implemented |
| **Position limits** | Per-strategy, per-symbol, per-account, portfolio-level | Basic (via user's custom RiskEngine) |
| **Exposure monitoring** | Real-time dashboard, alerts | None |

**Assessment:** PyEventBT has pre-trade risk (the `RiskEngine` in the Signal→Sizing→Risk→Order pipeline), which is the critical piece. But the industry separates pre-trade and post-trade risk:

- **Pre-trade** (PyEventBT's RiskEngine): "Should I place this order?" — must be synchronous, in-process
- **Post-trade**: "What is my total exposure across all strategies and accounts?" — can be async, separate process

**For your system:**
1. Keep pre-trade risk in the strategy core process (synchronous, no latency)
2. Add a post-trade risk service in the monitoring process that consumes FillEvents
3. Post-trade risk can enforce portfolio-level limits, correlation exposure, drawdown limits across strategies

---

## 32.3 What to Adopt from Each Architecture

### From LMAX
| Pattern | What to adopt | Priority |
|---|---|---|
| Single-threaded BLP | Already have this (TradingDirector) | Done |
| Event sourcing | Journal all input events for replay/recovery | High |
| Ring buffer (Disruptor) | Replace `queue.Queue` with lock-free ring buffer for live trading | Medium |
| Single Writer Principle | Replace SharedData with single-writer Cache | **Critical** |

### From NautilusTrader
| Pattern | What to adopt | Priority |
|---|---|---|
| MessageBus | Upgrade from simple queue to pub/sub + req/resp bus | High |
| Cache | Formal in-memory store replacing SharedData | **Critical** |
| Adapters | Formal adapter interface for brokers/data sources | High |
| Crash-only design | Idempotent operations, externalized state | Medium |

### From LEAN
| Pattern | What to adopt | Priority |
|---|---|---|
| Handler interfaces | Formalize all component interfaces (already started with IDataProvider, IExecutionEngine) | High |
| IResultHandler | Add a results/monitoring interface (missing in PyEventBT) | Medium |
| ISetupHandler | Add a setup/initialization interface | Low |
| TimeSlice | Consider unified data container instead of separate BarEvent per symbol | Low |

### From FIX Protocol
| Pattern | What to adopt | Priority |
|---|---|---|
| Message types | PyEventBT events already map to FIX (OrderEvent≈NewOrderSingle, FillEvent≈ExecutionReport) | Validated |
| Session management | Add connection lifecycle (Logon, Heartbeat, Logout) for broker adapters | Medium |
| Reject handling | Add OrderRejectEvent (FIX ExecutionReport with OrdStatus=Rejected) | High |

---

## 32.4 Prioritized Action Plan

Based on the comparison, here is the recommended order for building a new system:

### Phase 1 — Fix the Foundation (Critical)

| # | Action | Industry basis | Effort |
|---|---|---|---|
| 1 | Replace SharedData with single-writer Cache | LMAX SWP + NautilusTrader Cache | Medium |
| 2 | Remove Modules direct references — route queries through MessageBus | NautilusTrader MessageBus req/resp | High |
| 3 | Formalize adapter interfaces (IDataAdapter, IExecutionAdapter) | NautilusTrader Adapters + LEAN IBrokerage | Medium |

### Phase 2 — Add Industry Patterns (High)

| # | Action | Industry basis | Effort |
|---|---|---|---|
| 4 | Upgrade queue to MessageBus (pub/sub + req/resp + commands) | NautilusTrader MessageBus | High |
| 5 | Add event journaling for replay/recovery | LMAX event sourcing | Medium |
| 6 | Add OrderRejectEvent and proper error flow | FIX protocol reject handling | Low |
| 7 | Separate commands from events in type system | CQRS + NautilusTrader | Low |

### Phase 3 — Enable Distribution (Medium)

| # | Action | Industry basis | Effort |
|---|---|---|---|
| 8 | Add event serialization layer (JSON/Protobuf) | FIX, distributed systems | Medium |
| 9 | Split data ingestion into separate process (ZeroMQ) | Industry consensus | Medium |
| 10 | Split execution into separate process (ZeroMQ) | Industry consensus | Medium |
| 11 | Add monitoring/archival process (Kafka) | Industry consensus | Medium |

### Phase 4 — Production Hardening (Lower priority)

| # | Action | Industry basis | Effort |
|---|---|---|---|
| 12 | Add post-trade risk service | Industry separation of pre/post-trade | Medium |
| 13 | Add crash-only design (idempotent ops, externalized state) | NautilusTrader | High |
| 14 | Replace queue with ring buffer for live trading | LMAX Disruptor | High |
| 15 | Add multi-venue routing engine | NautilusTrader RoutingEngine | Medium |

---

## 32.5 Architecture Target — After All Phases

```
┌────────────────────────────────────────────────────────────────────────────┐
│                        STRATEGY CORE PROCESS (single-threaded)             │
│                                                                            │
│  ┌──────────┐    ┌──────────────────────────────────────────────────────┐  │
│  │  Event    │    │                   MessageBus                        │  │
│  │  Journal  │◀───│  pub/sub + req/resp + commands                      │  │
│  │  (append) │    │                                                      │  │
│  └──────────┘    │  ┌─────────┐ ┌────────┐ ┌──────┐ ┌──────────────┐  │  │
│                   │  │ Signal  │ │ Sizing │ │ Risk │ │ Schedule     │  │  │
│  ┌──────────┐    │  │ Engine  │ │ Engine │ │Engine│ │ Service      │  │  │
│  │  Cache    │    │  └─────────┘ └────────┘ └──────┘ └──────────────┘  │  │
│  │ (single   │◀──│                                                      │  │
│  │  writer)  │    │  ┌─────────────────────────────────────────────┐    │  │
│  └──────────┘    │  │          Event Loop (Kernel)                 │    │  │
│                   │  │  dequeue → dispatch → update Cache → journal │    │  │
│                   │  └─────────────────────────────────────────────┘    │  │
│                   └──────────────────────────────────────────────────────┘  │
└───────────┬──────────────────────────────────┬─────────────────────────────┘
            │ ZeroMQ                            │ ZeroMQ
            ▼                                   ▼
┌───────────────────┐                ┌───────────────────────┐
│ DATA PROCESS       │                │ EXECUTION PROCESS      │
│                    │                │                        │
│ IDataAdapter       │                │ IExecutionAdapter      │
│ ├── MT5Adapter     │                │ ├── MT5Adapter         │
│ ├── BinanceAdapter │                │ ├── IBKRAdapter        │
│ ├── CSVAdapter     │                │ ├── SimulatorAdapter   │
│ └── IBKRAdapter    │                │ └── BinanceAdapter     │
│                    │                │                        │
│ Normalization      │                │ Order routing          │
│ Heartbeat          │                │ Connection management  │
│ Reconnection       │                │ Retry / reconnection   │
└────────────────────┘                └───────────┬────────────┘
                                                  │ Kafka
                                                  ▼
                                      ┌───────────────────────┐
                                      │ MONITORING PROCESS     │
                                      │                        │
                                      │ Portfolio view         │
                                      │ Trade archiver         │
                                      │ Post-trade risk        │
                                      │ Performance dashboard  │
                                      │ Alert service          │
                                      └────────────────────────┘
```

**Key differences from PyEventBT:**
1. **Cache** replaces SharedData — single writer, read by all
2. **MessageBus** replaces queue.Queue — pub/sub + req/resp
3. **Event Journal** enables crash recovery and deterministic replay
4. **Adapters** replace connectors — formal interface, multi-venue capable
5. **Process separation** at I/O boundaries only — strategy core stays single-threaded

---

## 32.6 Key Insight

The industry research validates that PyEventBT got the **hardest part right**: the single-threaded event loop with typed events and sequential dispatch. This is the pattern that LMAX uses to process 6 million orders per second, and it's what NautilusTrader and LEAN both adopted.

The gaps are all in the **supporting infrastructure** around that loop:
- How state is stored (Cache vs SharedData)
- How components communicate (MessageBus vs direct calls)
- How brokers are abstracted (Adapters vs informal connectors)
- How failures are handled (event sourcing vs nothing)

The core loop stays. Everything around it gets upgraded.

---
