# Contract, Behavior & Protocol Diagrams

> Visual specification for every component in the target architecture. Each component is shown with its **Contract** (interface), **Behavior** (rules), and **Protocol** (communication). See also: [Implementation Guide](implementation_guide.md) | [Contracts & Protocols](contracts_protocols.md)

---

## Overview — All Components at a Glance

```mermaid
graph TB
    subgraph LEGEND["LEGEND"]
        direction LR
        C["Contract = Interface<br/>(what you implement)"]
        B["Behavior = Rules<br/>(how it must act)"]
        P["Protocol = Communication<br/>(who talks to whom)"]
    end

    subgraph SYSTEM["TARGET ARCHITECTURE"]
        DA[DataAdapter]
        K[Kernel]
        MB[MessageBus]
        CA[Cache]
        SE[SignalEngine]
        SZ[SizingEngine]
        RE[RiskEngine]
        EA[ExecutionAdapter]
        SC[StrategyContext]
        EJ[EventJournal]
        SM[SnapshotManager]
        TA[TradeArchiver]
        SS[ScheduleService]
    end

    DA -->|"BarEvent"| K
    K -->|"writes"| CA
    K -->|"publish/subscribe"| MB
    K -->|"record"| EJ
    K -->|"save"| SM
    K -->|"submit/close/cancel"| EA
    MB -->|"delivers events"| SE
    MB -->|"delivers events"| SZ
    MB -->|"delivers events"| RE
    MB -->|"delivers events"| TA
    MB -->|"delivers events"| SS
    SE -->|"reads"| SC
    SZ -->|"reads"| SC
    RE -->|"reads"| SC
    SC -->|"request"| MB
    SC -->|"reads"| CA

    style LEGEND fill:#f9f9f9,stroke:#ccc
    style SYSTEM fill:#f0f8ff,stroke:#4a90d9
```

---

## 1. Cache

```mermaid
graph LR
    subgraph CONTRACT["CONTRACT (Interface)"]
        direction TB
        R1["account: AccountState"]
        R2["get_symbol(symbol) → SymbolState"]
        R3["get_positions(strategy_id?, symbol?) → list"]
        R4["get_position_count(strategy_id?, symbol?, direction?) → int"]
        R5["get_pending_orders(strategy_id?, symbol?) → list"]
        W1["update_account(balance, equity, margin, margin_free)"]
        W2["update_position(snapshot)"]
        W3["remove_position(ticket)"]
        W4["set_symbol_info(symbol, info)"]
        W5["update_pending_order(snapshot)"]
        W6["remove_pending_order(ticket)"]
        W7["clear_all()"]
    end

    subgraph BEHAVIOR["BEHAVIOR (Rules)"]
        direction TB
        B1["Single Writer: ONLY Kernel writes"]
        B2["Write after event processing, not during"]
        B3["Instance, not global singleton"]
        B4["Read methods filter by strategy_id"]
        B5["Snapshots are immutable data objects"]
        B6["Account state refreshed after every fill"]
    end

    subgraph PROTOCOL["PROTOCOL (Communication)"]
        direction TB
        P1["Kernel ──writes──▶ Cache"]
        P2["SignalEngine ──reads──▶ Cache"]
        P3["SizingEngine ──reads──▶ Cache"]
        P4["RiskEngine ──reads──▶ Cache"]
        P5["No incoming events"]
        P6["No outgoing events"]
    end

    style CONTRACT fill:#e8f5e9,stroke:#4caf50
    style BEHAVIOR fill:#fff3e0,stroke:#ff9800
    style PROTOCOL fill:#e3f2fd,stroke:#2196f3
```

### Cache — Data Flow

```mermaid
sequenceDiagram
    participant EA as ExecutionAdapter
    participant K as Kernel
    participant C as Cache
    participant SE as SignalEngine
    participant SZ as SizingEngine

    Note over K: FillEvent received
    K->>EA: get_balance(), get_equity()
    EA-->>K: balance, equity, margin
    K->>C: update_position(snapshot)
    K->>C: update_account(balance, equity, ...)
    Note over C: State is now consistent

    Note over K: Next BarEvent dispatched
    K->>SE: on_bar(bar)
    SE->>C: cache.get_positions(strategy_id="1001")
    C-->>SE: [PositionSnapshot, ...]
    SE->>SZ: (signal produced → sizing)
    SZ->>C: cache.account.equity
    C-->>SZ: Decimal("10523.50")
```

---

## 2. MessageBus

```mermaid
graph LR
    subgraph CONTRACT["CONTRACT (Interface)"]
        direction TB
        C1["subscribe(topic, handler)"]
        C2["publish(topic, event)"]
        C3["dispatch_next() → bool"]
        C4["drain()"]
        C5["is_empty: bool"]
        C6["register_request_handler(topic, handler)"]
        C7["request(topic, **kwargs) → Any"]
    end

    subgraph BEHAVIOR["BEHAVIOR (Rules)"]
        direction TB
        B1["Single-threaded dispatch"]
        B2["Pub/sub: multiple subscribers per topic"]
        B3["Pub/sub: fire-and-forget (queued)"]
        B4["Request/response: one handler per topic"]
        B5["Request/response: synchronous (immediate)"]
        B6["Commands are queued like pub/sub"]
        B7["event.* = notification, command.* = mutation, request.* = query"]
        B8["No direct component references"]
    end

    subgraph PROTOCOL["PROTOCOL (Communication)"]
        direction TB
        P1["Kernel publishes events"]
        P2["Components subscribe to topics"]
        P3["StrategyContext sends requests"]
        P4["DataAdapter handles request responses"]
        P5["Kernel handles commands"]
    end

    style CONTRACT fill:#e8f5e9,stroke:#4caf50
    style BEHAVIOR fill:#fff3e0,stroke:#ff9800
    style PROTOCOL fill:#e3f2fd,stroke:#2196f3
```

### MessageBus — Three Communication Patterns

```mermaid
sequenceDiagram
    participant P as Producer
    participant BUS as MessageBus
    participant S1 as Subscriber A
    participant S2 as Subscriber B
    participant RH as Request Handler

    Note over P,S2: PATTERN 1: Pub/Sub (queued, multi-consumer)
    P->>BUS: publish("event.fill", fill)
    Note over BUS: Event queued
    BUS->>S1: handler_a(fill)
    BUS->>S2: handler_b(fill)

    Note over P,RH: PATTERN 2: Request/Response (immediate, single responder)
    P->>BUS: request("data.latest_bars", symbol="EURUSD", count=50)
    BUS->>RH: handler(symbol="EURUSD", count=50)
    RH-->>BUS: DataFrame
    BUS-->>P: DataFrame

    Note over P,S1: PATTERN 3: Command (queued, single consumer)
    P->>BUS: publish("command.close_position", {ticket: 42})
    Note over BUS: Command queued
    BUS->>S1: kernel._on_close_position({ticket: 42})
```

---

## 3. StrategyContext

```mermaid
graph LR
    subgraph CONTRACT["CONTRACT (Interface)"]
        direction TB
        C1["get_latest_bars(symbol, tf, count) → DataFrame"]
        C2["get_latest_tick(symbol) → dict"]
        C3["get_latest_bid(symbol) → Decimal"]
        C4["get_latest_ask(symbol) → Decimal"]
        C5["get_positions(symbol?) → list"]
        C6["get_position_count(symbol?, direction?) → int"]
        C7["get_pending_orders(symbol?) → list"]
        C8["get_account_balance() → Decimal"]
        C9["get_account_equity() → Decimal"]
        C10["close_position(ticket)"]
        C11["close_positions_by_symbol(symbol, direction?)"]
        C12["cancel_pending_order(ticket)"]
        C13["update_position_sl_tp(ticket, sl?, tp?)"]
    end

    subgraph BEHAVIOR["BEHAVIOR (Rules)"]
        direction TB
        B1["No direct references to other components"]
        B2["Data queries: synchronous via bus request"]
        B3["Portfolio queries: direct Cache read"]
        B4["Execution commands: async via bus publish"]
        B5["All queries scoped by strategy_id"]
    end

    subgraph PROTOCOL["PROTOCOL (Communication)"]
        direction TB
        P1["Data queries → bus.request() → DataAdapter"]
        P2["Portfolio queries → cache.get_*()"]
        P3["Execution commands → bus.publish(command.*)"]
    end

    style CONTRACT fill:#e8f5e9,stroke:#4caf50
    style BEHAVIOR fill:#fff3e0,stroke:#ff9800
    style PROTOCOL fill:#e3f2fd,stroke:#2196f3
```

### StrategyContext — Routing Diagram

```mermaid
flowchart LR
    UC["User Callback<br/>(signal/sizing/risk)"]

    subgraph CTX["StrategyContext"]
        DQ["Data Queries"]
        PQ["Portfolio Queries"]
        EC["Execution Commands"]
    end

    subgraph TARGETS["Targets"]
        BUS_REQ["MessageBus<br/>request()"]
        CACHE["Cache<br/>(read-only)"]
        BUS_PUB["MessageBus<br/>publish()"]
    end

    subgraph FINAL["Resolved By"]
        DA["DataAdapter"]
        CACHE2["Cache data"]
        KERNEL["Kernel<br/>(next dispatch)"]
    end

    UC --> DQ
    UC --> PQ
    UC --> EC

    DQ -->|"synchronous"| BUS_REQ
    PQ -->|"immediate"| CACHE
    EC -->|"queued"| BUS_PUB

    BUS_REQ --> DA
    CACHE --> CACHE2
    BUS_PUB --> KERNEL

    style CTX fill:#f3e5f5,stroke:#9c27b0
    style TARGETS fill:#f0f8ff,stroke:#4a90d9
    style FINAL fill:#f9f9f9,stroke:#999
```

---

## 4. Execution Adapter

```mermaid
graph LR
    subgraph CONTRACT["CONTRACT (Interface)"]
        direction TB
        C1["connect() → bool"]
        C2["disconnect()"]
        C3["submit_market_order(symbol, direction, volume, sl?, tp?, strategy_id?) → AdapterOrderResult"]
        C4["submit_limit_order(symbol, direction, volume, price, sl?, tp?, strategy_id?) → AdapterOrderResult"]
        C5["submit_stop_order(symbol, direction, volume, price, sl?, tp?, strategy_id?) → AdapterOrderResult"]
        C6["cancel_order(ticket) → AdapterOrderResult"]
        C7["close_position(ticket) → AdapterOrderResult"]
        C8["modify_position(ticket, sl?, tp?) → AdapterOrderResult"]
        C9["get_balance() → Decimal"]
        C10["get_equity() → Decimal"]
        C11["get_used_margin() → Decimal"]
        C12["get_free_margin() → Decimal"]
        C13["get_open_positions(strategy_id?) → list"]
        C14["get_pending_orders(strategy_id?) → list"]
        C15["get_symbol_info(symbol) → dict"]
    end

    subgraph BEHAVIOR["BEHAVIOR (Rules)"]
        direction TB
        B1["Standardized results: always AdapterOrderResult"]
        B2["Lifecycle: connect() before use, disconnect() after"]
        B3["Margin validation before execution"]
        B4["Strategy isolation via strategy_id"]
        B5["No event emission — returns to Kernel only"]
        B6["Idempotent close: already-closed → success=False"]
    end

    subgraph PROTOCOL["PROTOCOL (Communication)"]
        direction TB
        P1["Kernel → submit/close/cancel → Adapter"]
        P2["Adapter → AdapterOrderResult → Kernel"]
        P3["Kernel is the ONLY caller"]
        P4["Adapter never touches Bus or Cache"]
    end

    style CONTRACT fill:#e8f5e9,stroke:#4caf50
    style BEHAVIOR fill:#fff3e0,stroke:#ff9800
    style PROTOCOL fill:#e3f2fd,stroke:#2196f3
```

### Execution Adapter — Implementations & Routing

```mermaid
flowchart TB
    K["Kernel._on_order()"]

    subgraph ROUTING["RoutingExecutionAdapter"]
        R["_resolve(symbol)"]
    end

    subgraph ADAPTERS["Concrete Adapters"]
        SIM["SimulatorAdapter<br/>(backtest)"]
        MT5["MT5Adapter<br/>(live forex)"]
        IBKR["IBKRAdapter<br/>(live stocks)"]
        BIN["BinanceAdapter<br/>(live crypto)"]
    end

    K --> ROUTING
    R -->|"EUR*"| MT5
    R -->|"AAPL, MSFT"| IBKR
    R -->|"BTC*, ETH*"| BIN
    R -->|"backtest mode"| SIM

    MT5 -->|"AdapterOrderResult"| K
    IBKR -->|"AdapterOrderResult"| K
    BIN -->|"AdapterOrderResult"| K
    SIM -->|"AdapterOrderResult"| K

    style ROUTING fill:#fff9c4,stroke:#f9a825
    style ADAPTERS fill:#f0f8ff,stroke:#4a90d9
```

---

## 5. Data Adapter

```mermaid
graph LR
    subgraph CONTRACT["CONTRACT (Interface)"]
        direction TB
        C1["connect() → bool"]
        C2["disconnect()"]
        C3["subscribe(symbol, timeframe)"]
        C4["get_next_bar() → BarEvent | None"]
        C5["get_latest_bars(symbol, tf, count) → DataFrame"]
        C6["get_latest_tick(symbol) → dict"]
        C7["get_latest_bid(symbol) → Decimal"]
        C8["get_latest_ask(symbol) → Decimal"]
        C9["has_more_data: bool"]
    end

    subgraph BEHAVIOR["BEHAVIOR (Rules)"]
        direction TB
        B1["Chronological ordering"]
        B2["Multi-symbol alignment on same timestamp"]
        B3["No lookahead: only completed bars"]
        B4["has_more_data = False when exhausted"]
        B5["subscribe() before get_next_bar()"]
    end

    subgraph PROTOCOL["PROTOCOL (Communication)"]
        direction TB
        P1["Kernel → get_next_bar() → DataAdapter"]
        P2["DataAdapter → BarEvent → Kernel"]
        P3["StrategyContext → bus.request → get_latest_bars()"]
        P4["Kernel is caller for feed loop"]
        P5["Bus is caller for historical queries"]
    end

    style CONTRACT fill:#e8f5e9,stroke:#4caf50
    style BEHAVIOR fill:#fff3e0,stroke:#ff9800
    style PROTOCOL fill:#e3f2fd,stroke:#2196f3
```

### Data Adapter — Implementations

```mermaid
flowchart TB
    K["Kernel event loop"]

    subgraph ADAPTERS["Concrete Adapters"]
        CSV["CSVDataAdapter<br/>(backtest from files)"]
        MT5D["MT5DataAdapter<br/>(live MT5 terminal)"]
        BIND["BinanceDataAdapter<br/>(live WebSocket)"]
        COMP["CompositeDataAdapter<br/>(wraps multiple)"]
    end

    K -->|"get_next_bar()"| ADAPTERS
    ADAPTERS -->|"BarEvent"| K

    subgraph COMPOSITE["CompositeDataAdapter internals"]
        COMP2["Round-robin or<br/>earliest-timestamp-first"]
        CSV2["CSVDataAdapter<br/>(EURUSD)"]
        BIND2["BinanceDataAdapter<br/>(BTCUSDT)"]
    end

    COMP --> COMP2
    COMP2 --> CSV2
    COMP2 --> BIND2

    style ADAPTERS fill:#f0f8ff,stroke:#4a90d9
    style COMPOSITE fill:#f9f9f9,stroke:#999
```

---

## 6. Event Journal & Snapshot Manager

```mermaid
graph LR
    subgraph CONTRACT["CONTRACT (Interface)"]
        direction TB
        C1["Journal.record(topic, event) → int"]
        C2["Journal.replay(from_seq) → Iterator"]
        C3["Journal.close()"]
        C4["Snapshot.save(cache, journal_seq)"]
        C5["Snapshot.load_latest() → (data, seq)"]
        C6["Snapshot.restore_cache(cache, data)"]
    end

    subgraph BEHAVIOR["BEHAVIOR (Rules)"]
        direction TB
        B1["Append-only: never modify or delete"]
        B2["Journal BEFORE dispatch"]
        B3["Monotonic sequence numbers"]
        B4["Durable writes: flush + fsync"]
        B5["Snapshot every N bars"]
        B6["Recovery: load snapshot → replay remaining"]
        B7["Deterministic replay"]
    end

    subgraph PROTOCOL["PROTOCOL (Communication)"]
        direction TB
        P1["Kernel → record() → Journal (normal)"]
        P2["Journal → replay() → Kernel (recovery)"]
        P3["Kernel → save() → SnapshotManager"]
        P4["SnapshotManager → restore_cache() → Cache"]
        P5["Write-only in normal mode"]
        P6["Read-only in recovery mode"]
    end

    style CONTRACT fill:#e8f5e9,stroke:#4caf50
    style BEHAVIOR fill:#fff3e0,stroke:#ff9800
    style PROTOCOL fill:#e3f2fd,stroke:#2196f3
```

### Event Journal — Recovery Flow

```mermaid
sequenceDiagram
    participant SM as SnapshotManager
    participant K as Kernel
    participant C as Cache
    participant J as EventJournal
    participant BUS as MessageBus

    Note over K: STARTUP — Recovery phase
    K->>SM: load_latest()
    SM-->>K: (snapshot_data, seq=4500)

    K->>SM: restore_cache(cache, snapshot_data)
    SM->>C: update_account(...)
    SM->>C: update_position(...)
    Note over C: Cache restored to seq 4500

    K->>J: replay(from_seq=4500)
    loop For each entry after seq 4500
        J-->>K: {seq: 4501, topic: "event.bar", data: ...}
        K->>BUS: publish("event.bar", bar)
        BUS->>K: _on_bar(bar) → processes normally
        K->>C: update_account(...) / update_position(...)
    end
    Note over C: Cache fully recovered

    Note over K: NORMAL OPERATION begins
    K->>J: record("event.bar", new_bar)
    K->>BUS: publish("event.bar", new_bar)
```

---

## 7. Kernel

```mermaid
graph LR
    subgraph CONTRACT["CONTRACT (Interface)"]
        direction TB
        C1["bus: IMessageBus"]
        C2["cache: ICache (single writer)"]
        C3["journal: IEventJournal (optional)"]
        C4["run() → None"]
        C5["stop() → None"]
    end

    subgraph BEHAVIOR["BEHAVIOR (Rules)"]
        direction TB
        B1["Single-threaded: one event at a time"]
        B2["Single writer to Cache"]
        B3["Journal before dispatch (live mode)"]
        B4["Full processing before next event"]
        B5["BAR order: SL/TP → Schedule → Signal"]
        B6["SIGNAL chain: Size → Risk → Order (synchronous)"]
        B7["Commands queued, processed next cycle"]
        B8["Only Kernel calls adapters"]
        B9["Snapshot every N bars (live mode)"]
    end

    subgraph PROTOCOL["PROTOCOL (Communication)"]
        direction TB
        P1["DataAdapter → get_next_bar() → Kernel"]
        P2["Kernel → publish() → MessageBus"]
        P3["MessageBus → subscribe handlers → Kernel"]
        P4["Kernel → submit/close/cancel → ExecutionAdapter"]
        P5["Kernel → record() → EventJournal"]
        P6["Kernel → update/remove → Cache"]
    end

    style CONTRACT fill:#e8f5e9,stroke:#4caf50
    style BEHAVIOR fill:#fff3e0,stroke:#ff9800
    style PROTOCOL fill:#e3f2fd,stroke:#2196f3
```

### Kernel — Full Event Flow

```mermaid
sequenceDiagram
    participant DA as DataAdapter
    participant K as Kernel
    participant J as Journal
    participant BUS as MessageBus
    participant C as Cache
    participant SE as SignalEngine
    participant SZ as SizingEngine
    participant RE as RiskEngine
    participant EA as ExecutionAdapter
    participant TA as TradeArchiver

    Note over K: bus.is_empty = true
    K->>DA: get_next_bar()
    DA-->>K: BarEvent

    K->>J: record("event.bar", bar)
    K->>BUS: publish("event.bar", bar)

    Note over K: dispatch_next()
    BUS->>K: _on_bar(bar)
    K->>K: _check_sl_tp(bar)
    K->>SE: generate_signal(bar, ctx)
    SE->>C: get_positions(strategy_id)
    C-->>SE: []
    SE-->>K: SignalEvent

    K->>J: record("event.signal", signal)
    K->>BUS: publish("event.signal", signal)

    Note over K: dispatch_next()
    BUS->>K: _on_signal(signal)
    K->>SZ: get_suggested_order(signal, cache)
    SZ->>C: cache.account.equity
    C-->>SZ: Decimal("10523.50")
    SZ-->>K: SuggestedOrder(volume=0.1)

    K->>RE: assess_order(suggested, cache)
    RE-->>K: approved=True

    K->>J: record("event.order", order)
    K->>BUS: publish("event.order", order)

    Note over K: dispatch_next()
    BUS->>K: _on_order(order)
    K->>EA: submit_market_order(...)
    EA-->>K: AdapterOrderResult(success=True, ticket=42)

    K->>J: record("event.fill", fill)
    K->>BUS: publish("event.fill", fill)

    Note over K: dispatch_next()
    BUS->>K: _on_fill(fill)
    K->>C: update_position(snapshot)
    K->>EA: get_balance(), get_equity()
    EA-->>K: balance, equity
    K->>C: update_account(balance, equity, ...)
    BUS->>TA: archive(fill)
```

---

## 8. Signal Engine

```mermaid
graph LR
    subgraph CONTRACT["CONTRACT (Interface)"]
        direction TB
        C1["generate_signal(bar, ctx) → SignalEvent | list | None"]
        C2["strategy_id: str"]
    end

    subgraph BEHAVIOR["BEHAVIOR (Rules)"]
        direction TB
        B1["Pure analysis: read state, return signals"]
        B2["Filter on bar.timeframe if needed"]
        B3["Set strategy_id on every SignalEvent"]
        B4["Set sl ≠ 0 if RiskPctSizing downstream"]
        B5["Deterministic on replay"]
        B6["Side effects via commands only, not direct calls"]
    end

    subgraph PROTOCOL["PROTOCOL (Communication)"]
        direction TB
        P1["Bus → event.bar → SignalEngine"]
        P2["SignalEngine → SignalEvent → Bus"]
        P3["Reads: ctx.get_latest_bars() → Bus request"]
        P4["Reads: ctx.get_positions() → Cache"]
        P5["Mutations: ctx.close_positions() → Bus command"]
    end

    style CONTRACT fill:#e8f5e9,stroke:#4caf50
    style BEHAVIOR fill:#fff3e0,stroke:#ff9800
    style PROTOCOL fill:#e3f2fd,stroke:#2196f3
```

---

## 9. Sizing Engine

```mermaid
graph LR
    subgraph CONTRACT["CONTRACT (Interface)"]
        direction TB
        C1["get_suggested_order(signal, cache) → SuggestedOrder"]
    end

    subgraph BEHAVIOR["BEHAVIOR (Rules)"]
        direction TB
        B1["Volume respects symbol limits: min ≤ vol ≤ max, rounded to step"]
        B2["Volume = 0 means reject (insufficient equity)"]
        B3["Account currency conversion for risk % sizing"]
        B4["No side effects: read-only, returns a number"]
    end

    subgraph PROTOCOL["PROTOCOL (Communication)"]
        direction TB
        P1["Kernel._on_signal() calls get_suggested_order()"]
        P2["Reads: cache.account.equity"]
        P3["Reads: cache.get_symbol(symbol).volume_min"]
        P4["Returns: SuggestedOrder (not queued)"]
        P5["Synchronous within SIGNAL processing chain"]
    end

    style CONTRACT fill:#e8f5e9,stroke:#4caf50
    style BEHAVIOR fill:#fff3e0,stroke:#ff9800
    style PROTOCOL fill:#e3f2fd,stroke:#2196f3
```

---

## 10. Risk Engine

```mermaid
graph LR
    subgraph CONTRACT["CONTRACT (Interface)"]
        direction TB
        C1["assess_order(suggested_order, cache) → bool"]
    end

    subgraph BEHAVIOR["BEHAVIOR (Rules)"]
        direction TB
        B1["Gatekeeper: last checkpoint before execution"]
        B2["True = approve, False = reject"]
        B3["No side effects: read-only"]
        B4["Can read positions, equity, exposure from Cache"]
        B5["Composable: chain multiple risk checks"]
    end

    subgraph PROTOCOL["PROTOCOL (Communication)"]
        direction TB
        P1["Kernel._on_signal() calls assess_order()"]
        P2["Reads: cache.get_positions() for exposure"]
        P3["Reads: cache.account for equity/margin"]
        P4["Returns: bool (not queued)"]
        P5["Synchronous within SIGNAL processing chain"]
    end

    style CONTRACT fill:#e8f5e9,stroke:#4caf50
    style BEHAVIOR fill:#fff3e0,stroke:#ff9800
    style PROTOCOL fill:#e3f2fd,stroke:#2196f3
```

---

## 11. Trade Archiver

```mermaid
graph LR
    subgraph CONTRACT["CONTRACT (Interface)"]
        direction TB
        C1["archive(fill: FillEvent)"]
        C2["get_trades() → dict[int, FillEvent]"]
        C3["export_csv(path: str)"]
        C4["export_dataframe() → DataFrame"]
        C5["export_parquet(path: str)"]
    end

    subgraph BEHAVIOR["BEHAVIOR (Rules)"]
        direction TB
        B1["Append-only: never modify or delete"]
        B2["Archives both IN and OUT fills"]
        B3["Supports CSV, DataFrame, and Parquet export"]
    end

    subgraph PROTOCOL["PROTOCOL (Communication)"]
        direction TB
        P1["Bus → event.fill → TradeArchiver.archive()"]
        P2["TradeArchiver subscribes directly to event.fill"]
        P3["No outgoing events"]
        P4["Export methods called at end of backtest"]
    end

    style CONTRACT fill:#e8f5e9,stroke:#4caf50
    style BEHAVIOR fill:#fff3e0,stroke:#ff9800
    style PROTOCOL fill:#e3f2fd,stroke:#2196f3
```

---

## 12. Schedule Service

```mermaid
graph LR
    subgraph CONTRACT["CONTRACT (Interface)"]
        direction TB
        C1["register_callback(timeframe, callback)"]
        C2["on_bar(bar: BarEvent)"]
    end

    subgraph BEHAVIOR["BEHAVIOR (Rules)"]
        direction TB
        B1["Boundary detection: fire when bar crosses timeframe period"]
        B2["Callback signature: fn(event, ctx: StrategyContext)"]
        B3["Tracks last execution timestamp per callback"]
        B4["Multiple timeframes can coexist independently"]
    end

    subgraph PROTOCOL["PROTOCOL (Communication)"]
        direction TB
        P1["Bus → event.bar → ScheduleService.on_bar()"]
        P2["ScheduleService calls registered callbacks"]
        P3["Callbacks may return SignalEvents → Bus"]
        P4["Callbacks read via StrategyContext"]
    end

    style CONTRACT fill:#e8f5e9,stroke:#4caf50
    style BEHAVIOR fill:#fff3e0,stroke:#ff9800
    style PROTOCOL fill:#e3f2fd,stroke:#2196f3
```

---

## 13. Full System — Contract Boundaries

This diagram shows every contract boundary in the system. Each arrow crosses a contract — the sender and receiver can be implemented independently as long as both sides respect the contract, behavior, and protocol.

```mermaid
flowchart TB
    subgraph DATA["DATA PROCESS (future)"]
        DA["IDataAdapter"]
    end

    subgraph CORE["STRATEGY CORE PROCESS"]
        K["Kernel<br/>(IKernel)"]
        BUS["MessageBus<br/>(IMessageBus)"]
        CACHE["Cache<br/>(ICache + ICacheWriter)"]
        CTX["StrategyContext<br/>(IStrategyContext)"]
        SE["SignalEngine<br/>(user function)"]
        SZ["SizingEngine<br/>(ISizingEngine)"]
        RE["RiskEngine<br/>(IRiskEngine)"]
        SS["ScheduleService<br/>(IScheduleService)"]
        J["EventJournal<br/>(IEventJournal)"]
        SM["SnapshotManager<br/>(ISnapshotManager)"]
    end

    subgraph EXEC["EXECUTION PROCESS (future)"]
        EA["IExecutionAdapter"]
        ROUTE["RoutingAdapter"]
        MT5A["MT5Adapter"]
        IBKRA["IBKRAdapter"]
        BINA["BinanceAdapter"]
        SIMA["SimulatorAdapter"]
    end

    subgraph MON["MONITORING PROCESS (future)"]
        TA["TradeArchiver<br/>(ITradeArchiver)"]
        DASH["Dashboard"]
        RISK_POST["Post-Trade Risk"]
    end

    DA -->|"BarEvent<br/>(IDataAdapter contract)"| K
    K -->|"writes<br/>(ICacheWriter contract)"| CACHE
    K <-->|"pub/sub + req/resp<br/>(IMessageBus contract)"| BUS
    K -->|"record<br/>(IEventJournal contract)"| J
    K -->|"save<br/>(ISnapshotManager contract)"| SM
    SM -->|"restore"| CACHE

    BUS -->|"event.bar"| SE
    BUS -->|"event.bar"| SS
    SE -->|"reads<br/>(IStrategyContext contract)"| CTX
    CTX -->|"request"| BUS
    CTX -->|"reads<br/>(ICache contract)"| CACHE

    K -->|"calls<br/>(ISizingEngine contract)"| SZ
    SZ -->|"reads<br/>(ICache contract)"| CACHE
    K -->|"calls<br/>(IRiskEngine contract)"| RE
    RE -->|"reads<br/>(ICache contract)"| CACHE

    K -->|"submit/close/cancel<br/>(IExecutionAdapter contract)"| EA
    EA --> ROUTE
    ROUTE --> MT5A
    ROUTE --> IBKRA
    ROUTE --> BINA
    ROUTE --> SIMA

    BUS -->|"event.fill<br/>(ITradeArchiver contract)"| TA
    TA --> DASH
    TA --> RISK_POST

    style DATA fill:#e8f5e9,stroke:#4caf50
    style CORE fill:#f0f8ff,stroke:#4a90d9
    style EXEC fill:#fff3e0,stroke:#ff9800
    style MON fill:#f3e5f5,stroke:#9c27b0
```

---

## 14. Topic Map — All Bus Topics

| Topic | Type | Publisher | Subscriber(s) | Payload |
|---|---|---|---|---|
| `event.bar` | Event | Kernel | SignalEngine, ScheduleService | BarEvent |
| `event.signal` | Event | Kernel (from SignalEngine) | Kernel._on_signal | SignalEvent |
| `event.order` | Event | Kernel (from Risk approval) | Kernel._on_order | OrderEvent |
| `event.fill` | Event | Kernel (from ExecutionAdapter) | Kernel._on_fill, TradeArchiver | FillEvent |
| `command.close_position` | Command | StrategyContext | Kernel._on_close_position | {ticket, strategy_id} |
| `command.close_positions` | Command | StrategyContext | Kernel._on_close_positions | {symbol, direction?, strategy_id} |
| `command.cancel_order` | Command | StrategyContext | Kernel._on_cancel_order | {ticket, strategy_id} |
| `command.modify_position` | Command | StrategyContext | Kernel._on_modify_position | {ticket, sl?, tp?} |
| `request.data.latest_bars` | Request | StrategyContext | DataAdapter.get_latest_bars | {symbol, timeframe, count} → DataFrame |
| `request.data.latest_tick` | Request | StrategyContext | DataAdapter.get_latest_tick | {symbol} → dict |
| `request.data.latest_bid` | Request | StrategyContext | DataAdapter.get_latest_bid | {symbol} → Decimal |
| `request.data.latest_ask` | Request | StrategyContext | DataAdapter.get_latest_ask | {symbol} → Decimal |

---
