# Core Design Pattern — Understanding and Reusing the Event-Driven Architecture

> Extracted from the main documentation. See also: [Event Flow Diagrams](event_flow_diagram.md) | [Contracts & Protocols](contracts_protocols.md) | [Distributed Migration](distributed_migration.md)

This section explains the fundamental design pattern behind PyEventBT so that it can be understood independently and integrated into other projects.

### 27.1 The Pattern in One Sentence

All components communicate through a **single shared `Queue`** using **typed event objects** — no component ever calls another directly.

### 27.2 The Three Building Blocks

The entire architecture reduces to three pieces:

#### 1. Typed Events

Each event is a dataclass with a `type` field that identifies what kind of event it is:

| Event | Type Enum | Purpose |
|---|---|---|
| `BarEvent` | `EventType.BAR` | New market data arrived |
| `SignalEvent` | `EventType.SIGNAL` | A trade idea was generated |
| `OrderEvent` | `EventType.ORDER` | A sized and risk-approved instruction to trade |
| `FillEvent` | `EventType.FILL` | A trade was executed |
| `ScheduledEvent` | `EventType.SCHEDULED_EVENT` | A timed callback should fire |

Events are plain data — they carry all the information the next handler needs and nothing more.

#### 2. Independent Components (Producers and Consumers)

Each component receives the shared queue at construction time. It only interacts with other components by **consuming events** from the queue and **producing new events** back into it:

| Component | Consumes | Produces |
|---|---|---|
| `DataProvider` | *(external data)* | `BarEvent` |
| `SignalEngineService` | `BarEvent` | `SignalEvent` (or nothing) |
| `SizingEngineService` | `SignalEvent` | `SuggestedOrder` (intermediate, not queued) |
| `RiskEngineService` | `SuggestedOrder` | `OrderEvent` (or nothing, if rejected) |
| `ExecutionEngine` | `OrderEvent` | `FillEvent` |
| `PortfolioHandler` | `BarEvent`, `SignalEvent`, `FillEvent` | Orchestrates sizing → risk pipeline |
| `TradeArchiver` | `FillEvent` | *(persists trade history)* |
| `ScheduleService` | `BarEvent` | Fires registered callbacks at timeframe boundaries |

> **Note:** `SuggestedOrder` is the only intermediate entity that does not go through the queue. It is passed synchronously from the sizing engine to the risk engine within `PortfolioHandler.process_signal_event()`, because sizing and risk assessment are a single atomic step.

#### 3. The Event Loop (TradingDirector)

A single loop dequeues events one at a time and dispatches them to the correct handler:

```
while running:
    if queue is empty:
        data_provider.feed_next_bar()       # refill the queue
    else:
        event = queue.get()                  # dequeue one event
        handler_map[event.type](event)       # dispatch to the right handler
```

This is the only place where control flow decisions happen. Everything else is a handler that reacts to an event and optionally produces new ones.

### 27.3 Event Chain — How One Bar Becomes a Trade

```
  DataProvider
      │
      ▼
  BarEvent ──→ Queue
                 │
                 ▼
  TradingDirector dequeues BAR
      │
      ├─ PortfolioHandler.process_bar_event()
      │     └─ Portfolio updates: mark-to-market, check SL/TP hits
      │          └─ (may emit FillEvent if SL/TP triggered)
      │
      ├─ ScheduleService.run_scheduled_callbacks()
      │     └─ fires @run_every callbacks if a timeframe boundary crossed
      │
      └─ SignalEngineService.generate_signal()
            └─ calls user strategy logic
            └─ if signal: Queue.put(SignalEvent)
                              │
                              ▼
  TradingDirector dequeues SIGNAL
      │
      └─ PortfolioHandler.process_signal_event()
            ├─ SizingEngine  → SuggestedOrder (adds volume)
            └─ RiskEngine    → if approved: Queue.put(OrderEvent)
                                                │
                                                ▼
  TradingDirector dequeues ORDER
      │
      └─ ExecutionEngine._process_order_event()
            ├─ MARKET  → execute now    → Queue.put(FillEvent)
            └─ LIMIT/STOP → store pending (checked on future bars)
                                                │
                                                ▼
  TradingDirector dequeues FILL
      │
      └─ PortfolioHandler.process_fill_event()
            └─ TradeArchiver.archive_trade()
```

Each event is **fully processed** before the next one is dequeued. This guarantees deterministic, sequential execution with no race conditions.

### 27.4 Why This Pattern Works

| Property | How It Is Achieved |
|---|---|
| **Decoupling** | Components never import or call each other — they only know the queue and the event dataclasses |
| **Swappable engines** | Replace any engine (sizing, risk, execution, signal) without touching the rest — just register a different handler |
| **Backtest = Live** | The same queue, the same handlers, the same event types. Only two connectors change: `DataProvider` (CSV vs MT5 live) and `ExecutionEngine` (simulator vs real broker) |
| **Deterministic replay** | One event at a time, fully processed before the next. No concurrency, no race conditions |
| **Natural chaining** | BAR → SIGNAL → ORDER → FILL. Each handler produces the next event type in the chain |
| **Testability** | To test any component, put an event on a queue, call the handler, and inspect what new events were produced |

### 27.5 Integrating the Pattern Into Another Project

#### Minimal Skeleton

```python
from queue import Queue
from dataclasses import dataclass
from enum import Enum

# ── Step 1: Define your event types ──────────────────────────

class EventType(Enum):
    BAR = "BAR"
    SIGNAL = "SIGNAL"
    ORDER = "ORDER"
    FILL = "FILL"

@dataclass
class BarEvent:
    type: EventType = EventType.BAR
    symbol: str = ""
    close: float = 0.0

@dataclass
class SignalEvent:
    type: EventType = EventType.SIGNAL
    symbol: str = ""
    direction: str = ""  # "BUY" or "SELL"

@dataclass
class OrderEvent:
    type: EventType = EventType.ORDER
    symbol: str = ""
    direction: str = ""
    volume: float = 0.0

@dataclass
class FillEvent:
    type: EventType = EventType.FILL
    symbol: str = ""
    direction: str = ""
    volume: float = 0.0
    price: float = 0.0


# ── Step 2: Build independent components ─────────────────────

class SignalEngine:
    def __init__(self, queue: Queue):
        self.queue = queue

    def handle(self, bar: BarEvent):
        # Your alpha logic here
        if should_buy(bar):
            self.queue.put(SignalEvent(symbol=bar.symbol, direction="BUY"))


class SizingEngine:
    def handle(self, signal: SignalEvent) -> float:
        return 0.01  # fixed lot — replace with your sizing logic


class RiskEngine:
    def __init__(self, queue: Queue):
        self.queue = queue

    def handle(self, signal: SignalEvent, volume: float):
        if volume > 0:  # your risk filters here
            self.queue.put(OrderEvent(
                symbol=signal.symbol,
                direction=signal.direction,
                volume=volume,
            ))


class ExecutionEngine:
    def __init__(self, queue: Queue):
        self.queue = queue

    def handle(self, order: OrderEvent):
        # Execute against broker / simulator
        fill_price = execute(order)
        self.queue.put(FillEvent(
            symbol=order.symbol,
            direction=order.direction,
            volume=order.volume,
            price=fill_price,
        ))


class Portfolio:
    def record(self, fill: FillEvent):
        # Update positions, PnL, balance
        pass


# ── Step 3: Wire the event loop ──────────────────────────────

queue = Queue()
signal_engine = SignalEngine(queue)
sizing_engine = SizingEngine()
risk_engine = RiskEngine(queue)
execution_engine = ExecutionEngine(queue)
portfolio = Portfolio()

handlers = {
    EventType.BAR:    lambda e: signal_engine.handle(e),
    EventType.SIGNAL: lambda e: risk_engine.handle(e, sizing_engine.handle(e)),
    EventType.ORDER:  lambda e: execution_engine.handle(e),
    EventType.FILL:   lambda e: portfolio.record(e),
}

# ── Step 4: Run ──────────────────────────────────────────────

for bar in data_source:
    queue.put(bar)
    while not queue.empty():
        event = queue.get()
        handlers[event.type](event)
```

#### Customization Points

| What you want to change | What to modify |
|---|---|
| Add new event types (e.g. `REBALANCE`, `ALERT`) | Add an enum value, a dataclass, and a handler entry |
| Add or remove pipeline stages | Add/remove handler entries in the dispatch map |
| Switch data source (API, websocket, DB) | Replace the data feeding loop — everything downstream stays the same |
| Go async | Replace `queue.Queue` with `asyncio.Queue`, `while` with `async for` |
| Add logging / hooks | Wrap handlers or add hook calls before/after dispatch (see `HookService` in PyEventBT) |
| Support multiple strategies | Use `strategy_id` on events to route to the correct engine instances |

#### The Key Invariant to Preserve

**Components produce events; the loop dispatches them; no direct component-to-component calls.**

This single rule is what makes the architecture swappable, testable, and able to run the same code in backtest and live modes. As long as you preserve it, you can reshape everything else to fit your domain.

---
