# Architecture Limitations & Distributed Migration Guide

> Extracted from the main documentation. See also: [Design Pattern](design_pattern.md) | [Contracts & Protocols](contracts_protocols.md) | [Industry Research](industry_research.md)

This section documents the coupling points in PyEventBT that prevent running components as independent processes, and provides a concrete migration path for building a truly distributed version.

### 28.1 Current Coupling Points

While PyEventBT uses an event-driven pattern at the surface, three layers of tight coupling bind all components to a single process.

#### 28.1.1 `SharedData` — Global Mutable Singleton

**File:** `pyeventbt/broker/mt5_broker/shared/shared_data.py`

`SharedData` holds account balance, equity, margin, positions, and symbol info as **class-level (static) attributes**. Every component reads and writes it directly in shared memory:

```python
# ExecutionEngine writes (mt5_simulator_execution_engine_connector.py)
SharedData.account_info.__dict__["balance"] = self.balance
SharedData.account_info.__dict__["equity"] = self.equity
SharedData.account_info.__dict__["margin"] = self.used_margin
SharedData.account_info.__dict__["margin_free"] = self.free_margin

# SizingEngine reads (via mt5_simulator_connector.py → SharedData)
account_info = mt5.account_info()  # Returns SharedData.account_info
equity = Decimal(str(account_info.equity))
```

**Why this blocks distribution:** `SharedData` is process-local memory. If the ExecutionEngine runs in process A and the SizingEngine runs in process B, writes in A are invisible to B.

**Components that read SharedData:**

| Component | What it reads |
|---|---|
| SizingEngine (RiskPctSizing) | `account_info.equity` for position sizing |
| ExecutionEngine (simulator) | `account_info.balance`, `symbol_info` for margin calculations |
| Portfolio | Indirectly via ExecutionEngine balance/equity getters |
| DataProvider | `symbol_info[symbol].digits` for price encoding |

**Components that write SharedData:**

| Component | What it writes |
|---|---|
| ExecutionEngine (simulator) | `account_info.balance`, `.equity`, `.margin`, `.margin_free` |
| MT5 connector (initialize/login) | `account_info`, `terminal_info`, `symbol_info` |

#### 28.1.2 `Modules` — Direct Object References in User Callbacks

**File:** `pyeventbt/strategy/core/modules.py`

```python
class Modules(BaseModel):
    TRADING_CONTEXT: TypeContext
    DATA_PROVIDER: IDataProvider
    EXECUTION_ENGINE: IExecutionEngine
    PORTFOLIO: IPortfolio
```

Every user-defined callback (signal engines, sizing engines, risk engines, `@run_every` scheduled functions) receives `Modules` containing **live object references** to other components. User code calls methods on them synchronously:

```python
@strategy.custom_signal_engine(...)
def my_strategy(event: BarEvent, modules: Modules):
    # Synchronous read from DataProvider
    bars = modules.DATA_PROVIDER.get_latest_bars(symbol, tf, 50)

    # Synchronous read from Portfolio
    pos = modules.PORTFOLIO.get_number_of_strategy_open_positions_by_symbol(symbol)

    # Synchronous MUTATION of ExecutionEngine (bypasses the event queue entirely)
    modules.EXECUTION_ENGINE.close_strategy_short_positions_by_symbol(symbol)
```

**Why this blocks distribution:** These are in-process method calls that expect immediate return values. There is no serialization boundary, no network hop, and no way to intercept them without replacing the objects with RPC proxies.

#### 28.1.3 Synchronous Call Chains That Bypass the Queue

Several critical paths are direct method calls, not events:

```
Portfolio._update_portfolio(bar)
  └─→ ExecutionEngine._update_values_and_check_executions_and_fills(bar)  # sync call
  └─→ ExecutionEngine._get_strategy_positions()                           # sync call
  └─→ ExecutionEngine._get_account_balance()                              # sync call
  └─→ ExecutionEngine._get_account_equity()                               # sync call

PortfolioHandler.process_signal_event(signal)
  └─→ SizingEngine.get_suggested_order(signal)    # sync call, returns SuggestedOrder
  └─→ RiskEngine.assess_order(suggested_order)     # sync call, may queue OrderEvent

SizingEngine (RiskPctSizing).get_suggested_order(signal)
  └─→ DataProvider.get_latest_tick(symbol)          # sync call, needs current price
```

**Why this blocks distribution:** Each of these calls expects an immediate return value. The caller cannot proceed until the callee responds. In a distributed system, each of these becomes a network round-trip requiring request/response correlation.

### 28.2 Summary of Coupling by Component

| Component | SharedData | Modules refs | Sync calls to others | Queue usage |
|---|---|---|---|---|
| **DataProvider** | Reads `symbol_info` | — | — | Produces `BarEvent` |
| **SignalEngine** | — | Reads DataProvider, Portfolio; Mutates ExecutionEngine | 3+ sync calls per bar | Produces `SignalEvent` |
| **SizingEngine** | Reads `account_info` | Reads DataProvider | 1-2 sync calls per signal | Returns `SuggestedOrder` (not queued) |
| **RiskEngine** | — | Reads via Modules | — | Produces `OrderEvent` |
| **ExecutionEngine** | Reads + writes all fields | — | — | Produces `FillEvent` |
| **Portfolio** | — | — | 4+ sync calls to ExecutionEngine per bar | — |
| **PortfolioHandler** | — | — | Chains SizingEngine → RiskEngine synchronously | — |

### 28.3 Migration Path: From Single-Process to Distributed

#### Step 1: Eliminate `SharedData` — Replace with State Events

Introduce new event types that broadcast state changes:

```python
@dataclass
class AccountStateEvent:
    type: EventType = EventType.ACCOUNT_STATE
    balance: Decimal
    equity: Decimal
    margin: Decimal
    margin_free: Decimal
    timestamp: datetime

@dataclass
class SymbolInfoEvent:
    type: EventType = EventType.SYMBOL_INFO
    symbol: str
    digits: int
    volume_min: Decimal
    volume_max: Decimal
    point: Decimal
    # ... other fields
```

Each component subscribes to state topics and maintains a **local read-only cache**:

```python
class LocalStateCache:
    """Each process keeps its own copy, updated via events."""

    def __init__(self):
        self.account = AccountState()
        self.symbols = {}
        self.positions = {}

    def handle_account_state(self, event: AccountStateEvent):
        self.account.balance = event.balance
        self.account.equity = event.equity

    def handle_symbol_info(self, event: SymbolInfoEvent):
        self.symbols[event.symbol] = event
```

The ExecutionEngine publishes `AccountStateEvent` after every trade instead of mutating `SharedData`.

#### Step 2: Replace `Modules` Direct Calls with Command Events

Every synchronous call through `Modules` becomes a command/response event pair:

```python
# ── Commands (requests) ──────────────────────────────
@dataclass
class GetLatestBarsCommand:
    type: EventType = EventType.COMMAND
    request_id: str          # correlation ID
    reply_to: str            # response topic/queue
    symbol: str
    timeframe: str
    count: int

@dataclass
class ClosePositionsCommand:
    type: EventType = EventType.COMMAND
    request_id: str
    reply_to: str
    symbol: str
    direction: str           # "SHORT", "LONG", "ALL"

# ── Responses ────────────────────────────────────────
@dataclass
class GetLatestBarsResponse:
    type: EventType = EventType.RESPONSE
    request_id: str          # matches the command
    bars: list               # serialized bar data

@dataclass
class ClosePositionsResponse:
    type: EventType = EventType.RESPONSE
    request_id: str
    closed_count: int
    fills: list              # resulting FillEvents
```

The `Modules` object becomes a **thin async client** that sends commands and awaits responses:

```python
class DistributedModules:
    """Drop-in replacement for Modules that talks over message queues."""

    def __init__(self, command_queue, response_queue):
        self._cmd_q = command_queue
        self._resp_q = response_queue
        self._pending = {}

    async def get_latest_bars(self, symbol, tf, count):
        rid = str(uuid4())
        self._cmd_q.put(GetLatestBarsCommand(
            request_id=rid, reply_to="signal-engine-responses",
            symbol=symbol, timeframe=tf, count=count,
        ))
        return await self._wait_for(rid)

    async def close_short_positions(self, symbol):
        rid = str(uuid4())
        self._cmd_q.put(ClosePositionsCommand(
            request_id=rid, reply_to="signal-engine-responses",
            symbol=symbol, direction="SHORT",
        ))
        return await self._wait_for(rid)

    async def _wait_for(self, request_id, timeout=5.0):
        """Block until the matching response arrives."""
        # Implementation depends on transport (asyncio.Event, RabbitMQ consumer, etc.)
        ...
```

#### Step 3: Break the Sizing→Risk Synchronous Chain

Currently `PortfolioHandler.process_signal_event()` chains sizing and risk as direct calls. Make each step a queued event:

```
# Before (single process):
SIGNAL → [sync] SizingEngine.get_suggested_order() → [sync] RiskEngine.assess_order() → ORDER

# After (distributed):
SIGNAL → [queue: signals]
  → SizingEngine process consumes, produces → SuggestedOrderEvent → [queue: suggested-orders]
  → RiskEngine process consumes, produces   → OrderEvent          → [queue: orders]
```

This requires promoting `SuggestedOrder` from an internal entity to a first-class event:

```python
@dataclass
class SuggestedOrderEvent:
    type: EventType = EventType.SUGGESTED_ORDER
    symbol: str
    strategy_id: str
    signal_type: str
    volume: Decimal
    order_type: str
    order_price: Decimal
    sl: Decimal
    tp: Decimal
    buffer_data: dict
```

#### Step 4: Serialize Events for Wire Transfer

Current events use Python-specific types (`Decimal`, `datetime`, `Bar` dataclass). For cross-process transport, add a serialization layer:

```python
import json
from decimal import Decimal
from datetime import datetime

class EventSerializer:
    """JSON serializer for events crossing process boundaries."""

    @staticmethod
    def serialize(event) -> bytes:
        data = {
            "type": event.type.value,
            **{
                k: EventSerializer._encode(v)
                for k, v in event.__dict__.items() if k != "type"
            }
        }
        return json.dumps(data).encode()

    @staticmethod
    def _encode(value):
        if isinstance(value, Decimal):
            return str(value)
        if isinstance(value, datetime):
            return value.isoformat()
        return value

    @staticmethod
    def deserialize(raw: bytes, event_class):
        data = json.loads(raw)
        # Reconstruct types based on event_class field annotations
        ...
```

For production systems, consider Protocol Buffers or Apache Avro for schema evolution and type safety.

#### Step 5: Replace `queue.Queue` with a Message Broker

```python
# Before:
from queue import Queue
events_queue = Queue()
events_queue.put(BarEvent(...))
event = events_queue.get()

# After (RabbitMQ example):
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# Each event type gets its own topic/routing key
channel.exchange_declare(exchange='events', exchange_type='topic')

# Producer
channel.basic_publish(
    exchange='events',
    routing_key='event.bar',
    body=EventSerializer.serialize(bar_event),
)

# Consumer (in a separate process)
channel.queue_bind(queue='signal-engine', exchange='events', routing_key='event.bar')
channel.basic_consume(queue='signal-engine', on_message_callback=handle_bar)
```

### 28.4 Target Distributed Architecture

```
┌─────────────────┐     ┌─────────────────────────────────────────┐
│  Process 1       │     │            Message Broker                │
│  DataProvider    │────▶│  topic: event.bar                       │
│  (CSV/API/WS)   │     │  topic: event.signal                    │
└─────────────────┘     │  topic: event.suggested_order           │
                        │  topic: event.order                     │
┌─────────────────┐     │  topic: event.fill                      │
│  Process 2       │◀──▶│  topic: event.account_state             │
│  SignalEngine    │     │  topic: command.data_request            │
│  + LocalCache    │     │  topic: command.close_positions         │
└─────────────────┘     │  topic: response.*                      │
                        └──────────────┬──────────────────────────┘
┌─────────────────┐                    │
│  Process 3       │◀──────────────────┤
│  SizingEngine    │───────────────────┤
│  + LocalCache    │                   │
└─────────────────┘                    │
                                       │
┌─────────────────┐                    │
│  Process 4       │◀──────────────────┤
│  RiskEngine      │───────────────────┤
└─────────────────┘                    │
                                       │
┌─────────────────┐                    │
│  Process 5       │◀──────────────────┤
│  ExecutionEngine │───────────────────┤
│  (Sim / Broker)  │                   │
└─────────────────┘                    │
                                       │
┌─────────────────┐                    │
│  Process 6       │◀──────────────────┘
│  Portfolio +     │
│  TradeArchiver   │
└─────────────────┘
```

**Topic subscriptions per process:**

| Process | Subscribes to | Publishes to |
|---|---|---|
| DataProvider | *(external data source)* | `event.bar` |
| SignalEngine | `event.bar`, `event.account_state` | `event.signal`, `command.close_positions` |
| SizingEngine | `event.signal`, `event.account_state` | `event.suggested_order` |
| RiskEngine | `event.suggested_order` | `event.order` |
| ExecutionEngine | `event.order`, `command.close_positions` | `event.fill`, `event.account_state`, `response.*` |
| Portfolio | `event.bar`, `event.fill`, `event.account_state` | `event.account_state` (updated PnL) |

### 28.5 Trade-offs: Single-Process vs Distributed

| Aspect | Single-process (current) | Distributed (target) |
|---|---|---|
| **Latency** | Microseconds between components | Milliseconds (network hop per event) |
| **Consistency** | Immediate — all state is in-memory | Eventual — local caches may lag |
| **Complexity** | Low — one loop, one queue, one process | High — serialization, correlation IDs, timeouts, retries |
| **Debugging** | Simple stack traces | Distributed tracing (correlation IDs, timestamps) |
| **Scalability** | Single core | Horizontal — each component scales independently |
| **Fault isolation** | One crash kills everything | One process crash doesn't take down others |
| **Deployment** | Single binary / script | Multiple containers / services |
| **Backtest speed** | Fast (no I/O overhead) | Slower (serialization + network per event) |
| **User API** | Synchronous, simple callbacks | Async, requires `await` in user code |

### 28.6 Recommended Approach: Hybrid

For most trading systems, a practical middle ground is better than full distribution:

1. **Keep backtest single-process.** Speed matters; full distribution adds latency per bar with no benefit during replay.
2. **Distribute only for live trading.** Separate the DataProvider (market data) and ExecutionEngine (broker connection) into their own processes. These have genuine I/O boundaries.
3. **Keep SignalEngine + SizingEngine + RiskEngine in one process.** They form a tight computation pipeline where microsecond-level synchronous access to state (equity, positions, tick prices) is a real requirement for correct sizing and risk management.
4. **Use the message broker for the boundaries that matter:** data ingestion → strategy process → execution process → portfolio/monitoring process.

```
Process 1: DataProvider ──[broker]──▶ Process 2: Strategy      ──[broker]──▶ Process 3: Execution
(market data feed)                    (Signal + Sizing + Risk)                (broker connection)
                                              │                                       │
                                              └────────[broker]───────────────────────▶│
                                                                              Process 4: Portfolio
                                                                              (monitoring + archive)
```

This gives you fault isolation and independent scaling where it matters, without paying the complexity cost of fully decoupling components that genuinely need synchronous state access.

---
