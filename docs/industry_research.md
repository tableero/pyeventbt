# Industry Research — Event-Driven Trading Architectures & Process Separation

> Research conducted 2026-03-12. See also: [Design Pattern](design_pattern.md) | [Distributed Migration](distributed_migration.md) | [Contracts & Protocols](contracts_protocols.md)

This section consolidates findings from deep research into how the trading industry structures event-driven systems, how processes are separated, and how PyEventBT's architecture compares. All sources are linked at the end.

### 31.1 The Three Canonical Architectures

The trading industry has converged on three architectural patterns, each with different trade-offs:

#### 31.1.1 LMAX Architecture — Single-Threaded Event Sourcing

The [LMAX exchange](https://martinfowler.com/articles/lmax.html) processes 6 million orders per second on a **single thread**. Its architecture is the most referenced pattern for high-performance trading:

**Structure:**
- **Business Logic Processor (BLP):** Single-threaded Java process that holds ALL state in memory. Processes events sequentially. No database, no locks, no contention.
- **Input Disruptor:** Ring buffer that handles unmarshaling, journaling, and replication before events reach the BLP. Multiple concurrent tasks can process in parallel (unmarshal, journal, replicate) because they write to different slots.
- **Output Disruptors:** Marshal results for network transmission, organized by topic.

**Key insights for PyEventBT:**
- LMAX validates that **single-threaded, in-memory, sequential event processing** is the correct approach for trading. PyEventBT's `TradingDirector` loop follows this same pattern.
- State is recovered via **event replay** (event sourcing), not from a database. Current state is "entirely derivable by processing the input events."
- The Disruptor ring buffer replaces `queue.Queue` with a lock-free structure. For PyEventBT's single-threaded backtest, `queue.Queue` is sufficient. For live trading with I/O threads, a Disruptor-style ring buffer would be faster.
- LMAX does NOT split the business logic across processes. The BLP is deliberately a single process for determinism and performance.

**Relevance to process separation:** LMAX's answer is "don't separate the core logic." Keep signal generation, risk checks, and order creation in one single-threaded process. Only separate I/O (network, persistence) into different threads via Disruptors.

#### 31.1.2 QuantConnect LEAN — Handler-Based Pluggable Architecture

[QuantConnect's LEAN engine](https://www.quantconnect.com/docs/v2/lean-engine) is the most widely-used open-source trading engine (C# + Python):

**Structure:**
- **Engine class** coordinates between handler interfaces
- **IDataFeed:** Sources data from disk (backtest) or live streams (live). Produces `TimeSlice` objects.
- **ITransactionHandler:** Processes orders — either via fill models (backtest) or actual brokerage (live).
- **IRealTimeHandler:** Fires scheduled events. Mocked for backtest, real-time for live.
- **IResultHandler:** Collects and distributes results. Packets for backtest, streaming for live.
- **ISetupHandler:** Initializes algorithm, cash, portfolio state.

**Key insights for PyEventBT:**
- LEAN uses **interfaces for everything** — `IDataFeed`, `ITransactionHandler`, `IResultHandler`, `ISetupHandler`. This is the same pattern PyEventBT's `IDataProvider` and `IExecutionEngine` follow, but LEAN is more complete (PyEventBT is missing interfaces for results, setup, and scheduling).
- Backtest vs live is a **swap of handler implementations**, not separate code paths. Same pattern as PyEventBT.
- The `AlgorithmManager` contains the core time-step loop (equivalent to `TradingDirector`), processing `TimeSlice` objects.
- LEAN runs as a **single process** with handler abstraction. It does NOT distribute components across processes.

#### 31.1.3 NautilusTrader — Actor-Based with Ports and Adapters

[NautilusTrader](https://nautilustrader.io/docs/latest/concepts/architecture/) is the most architecturally advanced open-source trading framework (Rust core + Python API):

**Structure:**
- **NautilusKernel:** Central orchestrator. Single-threaded message dispatch (inspired by LMAX).
- **MessageBus:** Pub/sub + request/response + command/event patterns. Optionally backed by Redis for state persistence.
- **DataEngine:** Processes and routes market data to consumers based on subscriptions.
- **ExecutionEngine:** Manages order lifecycle, routes commands to adapters, coordinates with risk.
- **RiskEngine:** Pre-trade risk checks, position monitoring, configurable limits.
- **Cache:** In-memory storage for instruments, accounts, orders, positions.
- **Adapters:** Venue-specific implementations for data and execution (ports and adapters pattern).

**Key insights for PyEventBT:**
- NautilusTrader validates that a **MessageBus** (not just a queue) is the right abstraction for inter-component communication. The MessageBus supports pub/sub, request/response, and command patterns — PyEventBT only uses a simple queue.
- The **Cache** replaces PyEventBT's `SharedData` singleton. It's a proper in-memory store that components read from, rather than a globally-mutated singleton.
- **Adapters** (ports and adapters pattern) replace PyEventBT's connector pattern. Each venue (broker) has its own adapter implementing a standard interface. This is the industry standard for multi-broker support.
- The kernel is **single-threaded** for determinism. Same as LMAX and PyEventBT.
- NautilusTrader uses "crash-only design" — unified recovery paths, externalized state persistence, idempotent operations. PyEventBT has none of this.
- The same strategy codepath runs in both backtest and live with **no code changes** — exactly the goal PyEventBT shares.

### 31.2 How the Industry Splits Processes

Based on all sources researched, here is the consensus on what should and should not be separated:

#### What the Industry Keeps in One Process

| Component | Why it stays together |
|---|---|
| Signal generation + Sizing + Risk | These form a tight computation pipeline requiring microsecond access to current state (positions, equity, tick prices). Separating them adds latency that degrades sizing accuracy. |
| Event loop / Dispatcher | Must be single-threaded for determinism. The LMAX, LEAN, and NautilusTrader architectures all use a single-threaded core loop. |
| In-memory state (Cache) | Positions, orders, account balance must be immediately consistent for risk checks. Eventual consistency is unacceptable for pre-trade risk. |

#### What the Industry Separates

| Component | How it's separated | Why |
|---|---|---|
| **Market data ingestion** | Separate thread or process, feeds into main process via ring buffer or message bus | I/O bound (network), can be parallelized per venue |
| **Order execution / Broker connection** | Separate thread or adapter, receives commands from main process | I/O bound (network to broker), different reliability requirements |
| **Persistence / Journaling** | Separate thread that writes to disk/database asynchronously | I/O bound, must not block the main loop |
| **Results / Monitoring** | Separate process that subscribes to events | Read-only, can tolerate latency |
| **Risk monitoring (post-trade)** | Separate service that consumes fill events | Different update frequency than pre-trade risk |

#### The Critical Distinction: Pre-trade vs Post-trade

- **Pre-trade risk** (should I place this order?) must be synchronous, in-process, with immediate access to current state. This is what PyEventBT's `RiskEngine` does.
- **Post-trade risk** (what is my current exposure across all strategies?) can be a separate service consuming fill events asynchronously.

### 31.3 Communication Patterns Between Processes

#### 31.3.1 FIX Protocol — The Industry Standard

[FIX (Financial Information eXchange)](https://www.fixtrading.org/implementation-guide/) is the standard protocol for communication between trading components across process boundaries:

| Message Type | Purpose | PyEventBT Equivalent |
|---|---|---|
| `NewOrderSingle` (D) | Send a new order | `OrderEvent` |
| `ExecutionReport` (8) | Confirm order status (fill, cancel, reject) | `FillEvent` |
| `MarketDataRequest` (V) | Request market data subscription | N/A (DataProvider push model) |
| `MarketDataSnapshot` (W) | Market data update | `BarEvent` |
| `OrderCancelRequest` (F) | Cancel existing order | `ExecutionEngine.cancel_pending_order()` |

PyEventBT's event types map closely to FIX message types. If you were to distribute components, FIX (or a FIX-like schema) would be the natural wire format.

#### 31.3.2 Message Brokers

| Broker | Use Case | Latency | Throughput |
|---|---|---|---|
| **LMAX Disruptor** | Inter-thread (same process) | ~50 nanoseconds | 25M+ msg/sec |
| **Aeron** | Inter-process on same machine | ~1 microsecond | 10M+ msg/sec |
| **ZeroMQ** | Inter-process, same or different machines | ~10 microseconds | 1M+ msg/sec |
| **Kafka** | Cross-service, persistent, replayable | ~1-10 milliseconds | 100K+ msg/sec |
| **RabbitMQ** | Cross-service, routing, reliability | ~1-5 milliseconds | 50K+ msg/sec |

For a trading system, the recommendation is:
- **Within the strategy process:** In-memory queue or ring buffer (what PyEventBT uses)
- **Between data ingestion and strategy:** ZeroMQ or Aeron (low latency, no persistence needed)
- **Between strategy and monitoring/archival:** Kafka (persistence, replay, multiple consumers)

### 31.4 State Management Patterns

#### 31.4.1 Single Writer Principle

The industry consensus for trading systems is the **Single Writer Principle**: only one thread/process writes to any given data structure. Other threads read from it.

- LMAX: Single thread owns all business state. Input/output disruptors are read-only consumers.
- NautilusTrader: Kernel owns the Cache. Adapters publish events, kernel updates state.
- **PyEventBT's problem:** `SharedData` violates this — multiple components (ExecutionEngine, connectors) write to it.

#### 31.4.2 Event Sourcing

LMAX and NautilusTrader both use **event sourcing**: current state is derived from replaying the event log. This gives:
- Deterministic replay (backtest reproducibility)
- Crash recovery (replay events from last snapshot)
- Audit trail (every state change is an event)

PyEventBT does NOT use event sourcing — state is mutated in-place via `SharedData`. This is why crash recovery and replay are not possible.

#### 31.4.3 CQRS (Command Query Responsibility Segregation)

For distributed trading systems, [CQRS](https://dl.acm.org/doi/10.1145/3317614.3317632) separates:
- **Command side:** Accepts orders, modifies state (single writer)
- **Query side:** Read-only views of state (multiple readers, can be eventually consistent)

This maps to:
- **Command side:** ExecutionEngine (accepts OrderEvents, modifies positions)
- **Query side:** Portfolio, monitoring dashboards, risk reports (read positions, can lag slightly)

### 31.5 Framework Comparison — Where PyEventBT Stands

| Feature | PyEventBT | NautilusTrader | LEAN | Zipline | Backtrader |
|---|---|---|---|---|---|
| **Architecture** | Event-driven, single-thread loop | Event-driven, actor model, Rust core | Handler-based, C# core | Event-driven, Pipeline API | Event-driven, Cerebro orchestrator |
| **Event types** | BAR, SIGNAL, ORDER, FILL | Data, Order, Position, Account, Custom | TimeSlice (unified) | Bar-level events | Bar-level with notifications |
| **Inter-component comm** | `queue.Queue` | MessageBus (pub/sub + req/resp) | Interface-based handlers | Direct method calls | Direct method calls |
| **State management** | `SharedData` singleton (mutable) | Cache + event sourcing | In-memory with brokerage sync | In-memory | In-memory |
| **Broker abstraction** | 2 connectors (MT5 sim + live) | Adapter pattern, many venues | IBrokerage interface, many brokers | No live trading | Broker abstraction, few implementations |
| **Backtest = Live** | Same pipeline, different connectors | Same kernel, different adapters | Same interfaces, different handlers | Backtest only | Partial (live support limited) |
| **Multi-venue** | No | Yes (native) | Yes (native) | No | Limited |
| **Process separation** | Single process only | Single process, optional Redis bus | Single process only | Single process only | Single process only |
| **Live trading** | MT5 only (Windows) | Many brokers, cross-platform | Many brokers, cloud | No | Limited |
| **Maintenance (2026)** | Active | Active (Rust rewrite) | Active (cloud platform) | Community-maintained | Legacy/archive mode |

### 31.6 Key Takeaways for Building a Distributed Trading System

Based on all research, here are the validated recommendations:

#### 1. Keep the Core Single-Threaded

Every high-performance trading architecture (LMAX, NautilusTrader, LEAN) uses a **single-threaded core** for the business logic. This is not a limitation — it's a deliberate design choice for determinism, simplicity, and cache efficiency. PyEventBT's `TradingDirector` loop is correct in this regard.

#### 2. Separate I/O, Not Logic

The correct process boundary is between **computation** (signal + sizing + risk) and **I/O** (market data feeds, broker connections, persistence). Don't split the computation pipeline across processes.

#### 3. Replace SharedData with a Cache

NautilusTrader's `Cache` pattern is the right model: a single in-memory store, written by the kernel, read by all components. This replaces PyEventBT's `SharedData` singleton with a proper single-writer pattern.

#### 4. Use a MessageBus, Not Just a Queue

A `queue.Queue` is sufficient for a basic event loop, but a MessageBus with pub/sub, request/response, and command patterns (like NautilusTrader's) enables:
- Multiple consumers for the same event
- Request/response for synchronous queries
- Topic-based routing for multi-strategy systems

#### 5. Use Adapters for Multi-Broker Support

The "ports and adapters" (hexagonal architecture) pattern used by NautilusTrader is the industry standard for supporting multiple brokers. Each venue implements a standard adapter interface. This is a more formal version of what PyEventBT does with its connector pattern.

#### 6. Consider Event Sourcing for Recovery

If you need crash recovery or audit trails, event sourcing (LMAX pattern) is the answer: persist every input event to a journal, and recover by replaying from the last snapshot. This also gives you deterministic backtest replay for free.

#### 7. FIX-Like Schema for Wire Format

If you do distribute across processes, use a FIX-inspired message schema. PyEventBT's event types already map closely to FIX message types (BarEvent ≈ MarketData, OrderEvent ≈ NewOrderSingle, FillEvent ≈ ExecutionReport).

#### 8. Recommended Process Split for Live Trading

```
Process A: Market Data        Process B: Strategy Core         Process C: Execution
┌───────────────────┐        ┌─────────────────────────┐      ┌──────────────────┐
│ DataAdapter(s)    │        │ NautilusKernel-style     │      │ ExecAdapter(s)   │
│ - Binance WS      │──ZMQ──▶│ - MessageBus            │──ZMQ──▶│ - MT5 connector  │
│ - MT5 connector   │        │ - Cache (single writer)  │      │ - IBKR connector │
│ - CSV replay      │        │ - SignalEngine           │      │ - Binance connector│
└───────────────────┘        │ - SizingEngine           │      └────────┬─────────┘
                             │ - RiskEngine (pre-trade) │               │
                             │ - ScheduleService        │      ┌────────▼─────────┐
                             └────────────┬────────────┘      │ Process D:       │
                                          │                    │ Monitoring       │
                                          └────────Kafka──────▶│ - Portfolio view  │
                                                               │ - Trade archiver │
                                                               │ - Risk dashboard │
                                                               └──────────────────┘
```

### 31.7 Sources

- [The LMAX Architecture — Martin Fowler](https://martinfowler.com/articles/lmax.html)
- [LMAX Disruptor Technical Paper](https://lmax-exchange.github.io/disruptor/disruptor.html)
- [LMAX Disruptor as Architectural Pattern — Gokcer Belgusen](https://medium.com/garantibbva-teknoloji/lmax-disruptor-as-an-architectural-pattern-9719c803a1a5)
- [NautilusTrader Architecture Documentation](https://nautilustrader.io/docs/latest/concepts/architecture/)
- [NautilusTrader GitHub](https://github.com/nautechsystems/nautilus_trader)
- [QuantConnect LEAN Engine Documentation](https://www.quantconnect.com/docs/v2/lean-engine)
- [LEAN Engine & Execution Deep Dive — DeepWiki](https://deepwiki.com/QuantConnect/Lean/3-engine-and-execution)
- [Event-Driven Backtesting with Python — QuantStart](https://www.quantstart.com/articles/Event-Driven-Backtesting-with-Python-Part-I/)
- [Event-Driven Architecture for Trading Systems — The Full Stack](https://www.thefullstack.co.in/event-driven-architecture-trading-systems/)
- [The Python Backtesting Landscape (2026)](https://python.financial/)
- [FIX Protocol Implementation Guide — FIX Trading Community](https://www.fixtrading.org/implementation-guide/)
- [FIX Protocol Overview — ExtraHop](https://www.extrahop.com/resources/protocols/fix)
- [Event Sourcing and CQRS in Trading — ACM](https://dl.acm.org/doi/10.1145/3317614.3317632)
- [High Load Trading with Reveno CQRS/Event Sourcing — InfoQ](https://www.infoq.com/articles/High-load-transactions-Reveno-CQRS-Event-sourcing-framework/)
- [Scaling Trading Systems Without Sacrificing Consistency](https://moneysideoflife.com/2026/02/27/scaling-trading-systems-without-sacrificing-consistency/)
- [Data Pipeline Design in Algorithmic Trading — Edwin Salguero](https://medium.com/@edwinsalguero/data-pipeline-design-in-an-algorithmic-trading-system-ac0d8109c4b9)
- [Battle-Tested Backtesters: VectorBT vs Zipline vs Backtrader](https://medium.com/@trading.dude/battle-tested-backtesters-comparing-vectorbt-zipline-and-backtrader-for-financial-strategy-dee33d33a9e0)
- [Understanding LMAX Architecture — Farukh Mahammad](https://medium.com/@farukhmahammad199/understanding-lmax-architecture-a-high-performance-event-driven-system-beb8710a40cf)
