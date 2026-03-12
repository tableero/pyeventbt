# Implementation Guide — From Design to Working Code

> Concrete reference implementations for each architectural component. See also: [Architecture Comparison](architecture_comparison.md) | [Distributed Migration](distributed_migration.md) | [Contracts & Protocols](contracts_protocols.md)

This document bridges the gap between the architectural documentation (what to build and why) and the actual code (how to build it). Each section provides a working Python implementation that can be used as a starting point.

Every component is specified with three explicit layers:

- **Contract** — The interface. Method signatures, inputs, outputs. What you code against.
- **Behavior** — The rules. Invariants, ordering, side effects. How it must act at runtime.
- **Protocol** — The communication. Who sends what to whom, in what order. How components talk.

These three layers are what make components independently implementable, testable, and swappable. If you implement the contract, respect the behaviors, and follow the protocol, your component will work in the system without touching anything else.

---

## 33.1 Cache — Replaces `SharedData`

### Problem it solves

`SharedData` is a global mutable singleton with class-level attributes. Multiple components write to it via `__dict__` mutation. This violates the Single Writer Principle, makes distribution impossible, and prevents crash recovery.

### Contract

The Cache exposes two separate interfaces — a **read interface** for all components and a **write interface** exclusively for the Kernel:

```python
class ICache(Protocol):
    """Read interface — used by SignalEngine, SizingEngine, RiskEngine, StrategyContext."""

    @property
    def account(self) -> AccountState: ...
    def get_symbol(self, symbol: str) -> SymbolState | None: ...
    def get_positions(self, strategy_id: str = None, symbol: str = None) -> list[PositionSnapshot]: ...
    def get_position_count(self, strategy_id: str = None, symbol: str = None, direction: str = None) -> int: ...
    def get_pending_orders(self, strategy_id: str = None, symbol: str = None) -> list[PendingOrderSnapshot]: ...


class ICacheWriter(Protocol):
    """Write interface — ONLY used by the Kernel."""

    def update_account(self, balance: Decimal, equity: Decimal, margin: Decimal, margin_free: Decimal): ...
    def update_position(self, snapshot: PositionSnapshot): ...
    def remove_position(self, ticket: int): ...
    def set_symbol_info(self, symbol: str, info: SymbolState): ...
    def update_pending_order(self, snapshot: PendingOrderSnapshot): ...
    def remove_pending_order(self, ticket: int): ...
    def clear_all(self): ...
```

### Behavior

| Rule | Description |
|---|---|
| **Single Writer** | Only the Kernel writes to the Cache. All other components receive the read interface (`ICache`). This is the most important invariant. |
| **Write after event processing** | The Kernel updates the Cache after processing each FillEvent, not during. State is consistent between events. |
| **No global state** | Cache is an instance passed to components at construction, not a class with static attributes. |
| **Strategy isolation** | Read methods filter by `strategy_id` — a signal engine only sees its own positions. |
| **Snapshot semantics** | `PositionSnapshot` and `PendingOrderSnapshot` are immutable data objects. Components cannot modify Cache state by mutating a returned object. |
| **Account state always current** | After every fill, the Kernel refreshes balance/equity/margin from the execution adapter before any other event is processed. |

### Protocol

```
ExecutionAdapter ──(fill result)──▶ Kernel ──(writes)──▶ Cache
                                                           │
                                      ┌────────────────────┤ (reads)
                                      ▼                    ▼
                                SignalEngine          SizingEngine
                                (via StrategyContext)  (via Cache.account)
```

- **No incoming events.** The Cache does not subscribe to any topic.
- **No outgoing events.** The Cache does not publish anything.
- **Single direction:** Kernel writes → Components read. Never the reverse.

### Reference implementation

```python
from dataclasses import dataclass, field
from decimal import Decimal
from datetime import datetime
from typing import Optional


@dataclass
class AccountState:
    """Snapshot of the trading account. Updated after every fill."""
    balance: Decimal = Decimal("0")
    equity: Decimal = Decimal("0")
    margin: Decimal = Decimal("0")
    margin_free: Decimal = Decimal("0")
    currency: str = "USD"


@dataclass
class SymbolState:
    """Static properties for a tradeable instrument."""
    digits: int = 5
    volume_min: Decimal = Decimal("0.01")
    volume_max: Decimal = Decimal("100")
    volume_step: Decimal = Decimal("0.01")
    point: Decimal = Decimal("0.00001")
    trade_contract_size: Decimal = Decimal("100000")


@dataclass
class PositionSnapshot:
    """A single open position. Immutable from the reader's perspective."""
    ticket: int
    symbol: str
    direction: str          # "BUY" or "SELL"
    volume: Decimal
    price_entry: Decimal
    unrealized_pnl: Decimal
    strategy_id: str
    time_entry: datetime = None
    sl: Optional[Decimal] = None
    tp: Optional[Decimal] = None
    swap: Decimal = Decimal("0")


@dataclass
class PendingOrderSnapshot:
    """A single pending order."""
    ticket: int
    symbol: str
    order_type: str         # "BUY_LIMIT", "SELL_LIMIT", "BUY_STOP", "SELL_STOP"
    volume: Decimal
    price: Decimal
    strategy_id: str
    sl: Optional[Decimal] = None
    tp: Optional[Decimal] = None


class Cache:
    """
    Single-writer in-memory store.

    Write methods are prefixed with `update_` or `remove_` and are ONLY called
    by the Kernel after processing events.

    Read methods are properties or `get_` methods used by any component.
    """

    def __init__(self):
        self._account = AccountState()
        self._symbols: dict[str, SymbolState] = {}
        self._positions: dict[int, PositionSnapshot] = {}
        self._pending_orders: dict[int, PendingOrderSnapshot] = {}
        self._last_updated: datetime | None = None

    # ── Read interface ───────────────────────────────

    @property
    def account(self) -> AccountState:
        return self._account

    def get_symbol(self, symbol: str) -> SymbolState | None:
        return self._symbols.get(symbol)

    def get_positions(
        self,
        strategy_id: str = None,
        symbol: str = None,
    ) -> list[PositionSnapshot]:
        result = list(self._positions.values())
        if strategy_id:
            result = [p for p in result if p.strategy_id == strategy_id]
        if symbol:
            result = [p for p in result if p.symbol == symbol]
        return result

    def get_position_count(
        self,
        strategy_id: str = None,
        symbol: str = None,
        direction: str = None,
    ) -> int:
        positions = self.get_positions(strategy_id=strategy_id, symbol=symbol)
        if direction:
            positions = [p for p in positions if p.direction == direction]
        return len(positions)

    def get_pending_orders(
        self,
        strategy_id: str = None,
        symbol: str = None,
    ) -> list[PendingOrderSnapshot]:
        result = list(self._pending_orders.values())
        if strategy_id:
            result = [o for o in result if o.strategy_id == strategy_id]
        if symbol:
            result = [o for o in result if o.symbol == symbol]
        return result

    # ── Write interface (Kernel only) ────────────────

    def update_account(
        self,
        balance: Decimal,
        equity: Decimal,
        margin: Decimal,
        margin_free: Decimal,
    ):
        self._account.balance = balance
        self._account.equity = equity
        self._account.margin = margin
        self._account.margin_free = margin_free
        self._last_updated = datetime.now()

    def update_position(self, snapshot: PositionSnapshot):
        self._positions[snapshot.ticket] = snapshot

    def remove_position(self, ticket: int):
        self._positions.pop(ticket, None)

    def set_symbol_info(self, symbol: str, info: SymbolState):
        self._symbols[symbol] = info

    def update_pending_order(self, snapshot: PendingOrderSnapshot):
        self._pending_orders[snapshot.ticket] = snapshot

    def remove_pending_order(self, ticket: int):
        self._pending_orders.pop(ticket, None)

    def clear_all(self):
        """Reset for a new backtest run."""
        self._account = AccountState()
        self._positions.clear()
        self._pending_orders.clear()
        self._symbols.clear()
```

### How it plugs in

The Kernel is the single writer. After processing a `FillEvent`, it updates the Cache:

```python
# Inside the Kernel's _on_fill handler:
def _on_fill(self, fill: FillEvent):
    if fill.deal == DealType.IN:
        self.cache.update_position(PositionSnapshot(
            ticket=fill.position_id,
            symbol=fill.symbol,
            direction=fill.signal_type.value,
            volume=fill.volume,
            price_entry=fill.price,
            unrealized_pnl=Decimal("0"),
            strategy_id=fill.strategy_id,
        ))
    elif fill.deal == DealType.OUT:
        self.cache.remove_position(fill.position_id)

    self.cache.update_account(
        balance=self.exec_adapter.get_balance(),
        equity=self.exec_adapter.get_equity(),
        margin=self.exec_adapter.get_used_margin(),
        margin_free=self.exec_adapter.get_free_margin(),
    )
```

Components read from it — no direct calls to ExecutionEngine:

```python
# SizingEngine reads equity from Cache, not from ExecutionEngine
def get_suggested_order(self, signal: SignalEvent, cache: Cache) -> SuggestedOrder:
    equity = cache.account.equity
    risk_pct = Decimal("0.02")
    risk_amount = equity * risk_pct
    # ... calculate volume from risk_amount
```

### What changes from PyEventBT

| Before (SharedData) | After (Cache) |
|---|---|
| `SharedData.account_info.__dict__["balance"] = val` | `cache.update_account(balance=val, ...)` |
| `SharedData.account_info.equity` | `cache.account.equity` |
| `SharedData.symbol_info["EURUSD"].digits` | `cache.get_symbol("EURUSD").digits` |
| Written by ExecutionEngine + connectors | Written only by Kernel |
| Class-level static attributes | Instance with proper data structures |
| No filtering (all positions visible) | Filter by `strategy_id`, `symbol` |

---

## 33.2 MessageBus — Replaces `queue.Queue` + `Modules` Direct Calls

### Problem it solves

PyEventBT uses a simple `queue.Queue` with one handler per event type. Components that need data from other components bypass the queue entirely via `Modules` direct object references. This creates hidden dependencies and blocks distribution.

### Contract

```python
class IMessageBus(Protocol):
    """Central communication hub. All inter-component communication goes through here."""

    # ── Pub/Sub ──
    def subscribe(self, topic: str, handler: Callable) -> None:
        """Register a handler for a topic. Multiple handlers per topic allowed."""

    def publish(self, topic: str, event) -> None:
        """Enqueue an event for delivery to all subscribers of that topic."""

    def dispatch_next(self) -> bool:
        """Dequeue one event, call all subscribers. Returns False if empty."""

    def drain(self) -> None:
        """Process all queued events until empty."""

    @property
    def is_empty(self) -> bool: ...

    # ── Request/Response ──
    def register_request_handler(self, topic: str, handler: Callable) -> None:
        """Register a handler for synchronous queries. One handler per topic."""

    def request(self, topic: str, **kwargs) -> Any:
        """Send a synchronous request, return the response immediately."""
```

### Behavior

| Rule | Description |
|---|---|
| **Single-threaded dispatch** | All event dispatch happens on one thread. No locks, no async, no concurrency. |
| **Pub/sub: multiple subscribers** | Multiple handlers can subscribe to the same topic. All are called in registration order when an event is dispatched. |
| **Pub/sub: fire-and-forget** | `publish()` enqueues the event and returns immediately. Handlers are called during `dispatch_next()`. |
| **Request/response: single handler** | Only one handler per request topic. Registering a second raises an error. |
| **Request/response: synchronous** | `request()` calls the handler immediately and returns its result. No queueing. |
| **Commands are published** | Mutation requests (close position, cancel order) are published as events with `command.*` topics. They are queued and dispatched like any other event. |
| **Events vs Commands naming** | Topics starting with `event.*` are notifications (something happened). Topics starting with `command.*` are requests (do something). Topics starting with `request.*` are synchronous queries (tell me something). |
| **No direct references** | Components never hold references to other components. All communication goes through the bus. |

### Protocol

Three communication patterns, each with different semantics:

```
1. PUB/SUB (fire-and-forget, multiple consumers)
   ────────────────────────────────────────────
   Producer ──publish("event.bar", bar)──▶ Bus queue
                                            │
                           dispatch_next()  │
                                            ├──▶ Subscriber A (portfolio)
                                            ├──▶ Subscriber B (schedule)
                                            └──▶ Subscriber C (signal engine)

2. REQUEST/RESPONSE (synchronous query, single responder)
   ────────────────────────────────────────────────────────
   StrategyContext ──request("data.latest_bars", symbol=..., ...)──▶ Bus
                                                                      │
                                                           (immediate call)
                                                                      │
                   ◀──return DataFrame──────────────────── DataAdapter handler

3. COMMAND (mutation request, queued like pub/sub)
   ──────────────────────────────────────────────
   StrategyContext ──publish("command.close_positions", {...})──▶ Bus queue
                                                                    │
                                                       dispatch_next()
                                                                    │
                                                                    └──▶ Kernel._on_close_positions()
```

### Reference implementation

```python
from collections import defaultdict
from queue import Queue
from typing import Callable, Any
from uuid import uuid4


class MessageBus:
    """
    Pub/sub + request/response message bus.

    Three communication patterns:
    1. publish/subscribe — fire-and-forget, multiple consumers
    2. request/response — synchronous query, single responder
    3. command — mutation request, processed as a queued event

    All patterns are single-threaded. No locks, no async.
    """

    def __init__(self):
        self._subscribers: dict[str, list[Callable]] = defaultdict(list)
        self._queue: Queue = Queue()
        self._request_handlers: dict[str, Callable] = {}

    # ── Pub/Sub ──────────────────────────────────────

    def subscribe(self, topic: str, handler: Callable):
        """Register a handler for a topic. Multiple handlers per topic allowed."""
        self._subscribers[topic].append(handler)

    def publish(self, topic: str, event):
        """Enqueue an event. Will be dispatched to all subscribers on next dispatch cycle."""
        self._queue.put((topic, event))

    def dispatch_next(self) -> bool:
        """
        Dequeue one event and call all subscribers for its topic.
        Returns False if the queue is empty.
        """
        if self._queue.empty():
            return False
        topic, event = self._queue.get()
        for handler in self._subscribers.get(topic, []):
            handler(event)
        return True

    def drain(self):
        """Process all queued events until the queue is empty."""
        while self.dispatch_next():
            pass

    @property
    def is_empty(self) -> bool:
        return self._queue.empty()

    # ── Request/Response ─────────────────────────────

    def register_request_handler(self, topic: str, handler: Callable):
        """
        Register a handler that responds to synchronous requests.
        Only one handler per request topic (unlike pub/sub).
        """
        if topic in self._request_handlers:
            raise ValueError(f"Request handler already registered for: {topic}")
        self._request_handlers[topic] = handler

    def request(self, topic: str, **kwargs) -> Any:
        """
        Send a synchronous request and return the response.

        This is the replacement for Modules direct calls:
        - Before: modules.DATA_PROVIDER.get_latest_bars(symbol, tf, 50)
        - After:  bus.request("data.latest_bars", symbol=symbol, timeframe=tf, count=50)

        Single-threaded: the handler is called immediately and its return value is passed back.
        """
        handler = self._request_handlers.get(topic)
        if handler is None:
            raise ValueError(f"No request handler registered for: {topic}")
        return handler(**kwargs)
```

### How it plugs in — wiring

```python
bus = MessageBus()

# ── Pub/sub subscriptions (replaces TradingDirector handler map) ──

# BAR events go to multiple consumers (not possible with current queue.Queue)
bus.subscribe("event.bar", kernel.on_bar)

# SIGNAL, ORDER, FILL each have their handler
bus.subscribe("event.signal", kernel.on_signal)
bus.subscribe("event.order", kernel.on_order)
bus.subscribe("event.fill", kernel.on_fill)

# Additional subscribers — archiver listens to fills directly
bus.subscribe("event.fill", trade_archiver.archive)

# Command subscribers
bus.subscribe("command.close_positions", kernel.on_close_command)
bus.subscribe("command.cancel_order", kernel.on_cancel_command)

# ── Request handlers (replaces Modules direct calls) ──

bus.register_request_handler(
    "data.latest_bars",
    lambda symbol, timeframe, count: data_adapter.get_latest_bars(symbol, timeframe, count),
)
bus.register_request_handler(
    "data.latest_tick",
    lambda symbol: data_adapter.get_latest_tick(symbol),
)
bus.register_request_handler(
    "data.latest_bid",
    lambda symbol: data_adapter.get_latest_bid(symbol),
)
bus.register_request_handler(
    "data.latest_ask",
    lambda symbol: data_adapter.get_latest_ask(symbol),
)
```

### StrategyContext — replaces `Modules`

User callbacks currently receive `Modules` with direct object references. The replacement is `StrategyContext`, a thin wrapper around the bus and cache.

#### Contract

```python
class IStrategyContext(Protocol):
    """Passed to every user callback. The ONLY way user code interacts with the system."""

    # ── Data queries ──
    def get_latest_bars(self, symbol: str, timeframe: str, count: int) -> pl.DataFrame: ...
    def get_latest_tick(self, symbol: str) -> dict: ...
    def get_latest_bid(self, symbol: str) -> Decimal: ...
    def get_latest_ask(self, symbol: str) -> Decimal: ...

    # ── Portfolio queries ──
    def get_positions(self, symbol: str = None) -> list[PositionSnapshot]: ...
    def get_position_count(self, symbol: str = None, direction: str = None) -> int: ...
    def get_pending_orders(self, symbol: str = None) -> list[PendingOrderSnapshot]: ...
    def get_account_balance(self) -> Decimal: ...
    def get_account_equity(self) -> Decimal: ...

    # ── Execution commands ──
    def close_position(self, ticket: int) -> None: ...
    def close_positions_by_symbol(self, symbol: str, direction: str = None) -> None: ...
    def cancel_pending_order(self, ticket: int) -> None: ...
    def update_position_sl_tp(self, ticket: int, sl: Decimal = None, tp: Decimal = None) -> None: ...
```

#### Behavior

| Rule | Description |
|---|---|
| **No direct references** | User code never gets a reference to DataProvider, ExecutionEngine, or Portfolio. Only StrategyContext methods. |
| **Data queries are synchronous** | `get_latest_bars()`, `get_latest_tick()` go through the bus's request/response — immediate return. |
| **Portfolio queries read Cache** | `get_positions()`, `get_account_equity()` read directly from the Cache. No bus round-trip needed. |
| **Execution commands are async** | `close_position()`, `cancel_pending_order()` publish command events to the bus. They are processed on the next dispatch cycle, not immediately. |
| **Strategy-scoped** | All queries are automatically filtered by the context's `strategy_id`. A signal engine only sees its own positions. |

#### Protocol

```
User callback (signal/sizing/risk/scheduled)
    │
    ├── ctx.get_latest_bars()     ──▶ bus.request("data.latest_bars")     ──▶ DataAdapter (immediate)
    ├── ctx.get_positions()       ──▶ cache.get_positions(strategy_id=..) ──▶ Cache (immediate)
    └── ctx.close_position()      ──▶ bus.publish("command.close_position") ──▶ queued for Kernel
```

#### Reference implementation

```python
from dataclasses import dataclass
from typing import Any


@dataclass
class StrategyContext:
    """
    Passed to every user callback (signal engine, sizing engine, risk engine,
    scheduled callbacks, hooks).

    Replaces Modules. All reads go through Cache, all queries go through
    the MessageBus, all mutations are published as command events.

    The user never gets a direct reference to DataProvider, ExecutionEngine,
    or Portfolio.
    """
    bus: MessageBus
    cache: Cache
    strategy_id: str
    trading_context: str    # "BACKTEST" or "LIVE"

    # ── Data queries (replace modules.DATA_PROVIDER.*) ──

    def get_latest_bars(self, symbol: str, timeframe: str, count: int):
        return self.bus.request("data.latest_bars",
                                symbol=symbol, timeframe=timeframe, count=count)

    def get_latest_tick(self, symbol: str) -> dict:
        return self.bus.request("data.latest_tick", symbol=symbol)

    def get_latest_bid(self, symbol: str):
        return self.bus.request("data.latest_bid", symbol=symbol)

    def get_latest_ask(self, symbol: str):
        return self.bus.request("data.latest_ask", symbol=symbol)

    # ── Portfolio queries (replace modules.PORTFOLIO.*) ──

    def get_positions(self, symbol: str = None) -> list[PositionSnapshot]:
        return self.cache.get_positions(strategy_id=self.strategy_id, symbol=symbol)

    def get_position_count(self, symbol: str = None, direction: str = None) -> int:
        return self.cache.get_position_count(
            strategy_id=self.strategy_id, symbol=symbol, direction=direction)

    def get_pending_orders(self, symbol: str = None) -> list[PendingOrderSnapshot]:
        return self.cache.get_pending_orders(strategy_id=self.strategy_id, symbol=symbol)

    def get_account_balance(self):
        return self.cache.account.balance

    def get_account_equity(self):
        return self.cache.account.equity

    # ── Execution commands (replace modules.EXECUTION_ENGINE.*) ──

    def close_position(self, ticket: int):
        self.bus.publish("command.close_position", {
            "ticket": ticket,
            "strategy_id": self.strategy_id,
        })

    def close_positions_by_symbol(self, symbol: str, direction: str = None):
        self.bus.publish("command.close_positions", {
            "symbol": symbol,
            "direction": direction,
            "strategy_id": self.strategy_id,
        })

    def cancel_pending_order(self, ticket: int):
        self.bus.publish("command.cancel_order", {
            "ticket": ticket,
            "strategy_id": self.strategy_id,
        })

    def update_position_sl_tp(self, ticket: int, sl=None, tp=None):
        self.bus.publish("command.modify_position", {
            "ticket": ticket,
            "sl": sl,
            "tp": tp,
            "strategy_id": self.strategy_id,
        })
```

### What user code looks like before vs after

```python
# ── BEFORE (PyEventBT with Modules) ─────────────────────────

@strategy.custom_signal_engine(strategy_id="1001", strategy_timeframes=[...])
def my_strategy(event: BarEvent, modules: Modules):
    # Direct call to DataProvider object
    bars = modules.DATA_PROVIDER.get_latest_bars("EURUSD", "1h", 50)
    closes = bars.select("close").to_numpy().flatten()

    # Direct call to Portfolio object
    pos = modules.PORTFOLIO.get_number_of_strategy_open_positions_by_symbol("EURUSD")

    # Direct mutation of ExecutionEngine object
    if some_condition:
        modules.EXECUTION_ENGINE.close_strategy_short_positions_by_symbol("EURUSD")

    return SignalEvent(...)


# ── AFTER (with StrategyContext) ─────────────────────────────

@strategy.custom_signal_engine(strategy_id="1001", strategy_timeframes=[...])
def my_strategy(event: BarEvent, ctx: StrategyContext):
    # Query goes through MessageBus
    bars = ctx.get_latest_bars("EURUSD", "1h", 50)
    closes = bars.select("close").to_numpy().flatten()

    # Read goes through Cache
    pos_count = ctx.get_position_count(symbol="EURUSD")

    # Mutation published as command event (processed by Kernel, not by user code)
    if some_condition:
        ctx.close_positions_by_symbol("EURUSD", direction="SELL")

    return SignalEvent(...)
```

### What changes from PyEventBT

| Before (queue.Queue + Modules) | After (MessageBus + StrategyContext) |
|---|---|
| `events_queue = Queue()` | `bus = MessageBus()` |
| `events_queue.put(event)` | `bus.publish("event.bar", event)` |
| One handler per event type | Multiple subscribers per topic |
| `modules.DATA_PROVIDER.get_latest_bars(...)` | `ctx.get_latest_bars(...)` → bus request |
| `modules.EXECUTION_ENGINE.close_positions(...)` | `ctx.close_positions_by_symbol(...)` → bus command |
| `modules.PORTFOLIO.get_positions(...)` | `ctx.get_positions(...)` → cache read |
| Hidden dependencies (who calls whom?) | Explicit topics (who subscribes to what?) |

---

## 33.3 Adapters — Replaces Informal Connectors

### Problem it solves

PyEventBT has two hardcoded connectors (`Mt5SimulatorExecutionEngineConnector` and `Mt5LiveExecutionEngineConnector`). There is no formal interface that a new broker must implement, no routing for multi-broker setups, and no separation between data adapters and execution adapters.

### Contract

Two separate adapter contracts — one for market data, one for order execution — because a system may use different sources for each (e.g., Binance data + MT5 execution).

Every method is `@abstractmethod`. If it's in the contract, you must implement it.

#### Behavior — Execution Adapter

| Rule | Description |
|---|---|
| **Standardized results** | Every method returns `AdapterOrderResult` regardless of broker. The Kernel never sees broker-specific types. |
| **Lifecycle management** | `connect()` must be called before any other method. `disconnect()` must release all resources. |
| **Margin validation** | `submit_market_order()` must check available margin before execution. Return `success=False` if insufficient. |
| **Strategy isolation** | `strategy_id` is passed with every order. Operations like `get_open_positions(strategy_id=...)` must filter by it. |
| **No event emission** | Adapters do NOT publish events to the bus. They return results to the Kernel, which is responsible for publishing FillEvents. |
| **Idempotent close** | `close_position(ticket)` for an already-closed position returns `success=False`, not an exception. |

#### Behavior — Data Adapter

| Rule | Description |
|---|---|
| **Chronological ordering** | `get_next_bar()` must return bars in chronological order. |
| **Multi-symbol alignment** | When multiple symbols are subscribed, all bars for the same timestamp must be returned before advancing. |
| **No lookahead** | `get_latest_bars()` must return only completed bars, never the currently forming bar. |
| **End-of-data signal** | `has_more_data` must return `False` when backtest data is exhausted. Always `True` for live adapters. |
| **Subscription required** | `subscribe(symbol, timeframe)` must be called before `get_next_bar()` returns bars for that pair. |

#### Protocol — Execution Adapter

```
Kernel ──submit_market_order(...)──▶ ExecutionAdapter ──▶ Broker API
Kernel ◀──AdapterOrderResult──────── ExecutionAdapter ◀── Broker response

Kernel ──close_position(ticket)───▶ ExecutionAdapter ──▶ Broker API
Kernel ◀──AdapterOrderResult──────── ExecutionAdapter ◀── Broker response

The Kernel is the ONLY caller. No other component interacts with the adapter directly.
```

#### Protocol — Data Adapter

```
Kernel ──get_next_bar()───────────▶ DataAdapter ──▶ CSV / API / WebSocket
Kernel ◀──BarEvent | None────────── DataAdapter ◀── Data source

StrategyContext ──(via bus request)──▶ DataAdapter.get_latest_bars()
StrategyContext ◀──DataFrame────────── DataAdapter

The Kernel calls get_next_bar() to feed the loop.
StrategyContext calls get_latest_bars() via bus request for historical lookback.
```

### Reference implementation — Execution Adapter

```python
from abc import ABC, abstractmethod
from decimal import Decimal
from dataclasses import dataclass
from typing import Optional


@dataclass
class AdapterOrderResult:
    """Standardized result from any broker. Same structure regardless of venue."""
    success: bool
    order_ticket: int | None = None
    fill_price: Decimal | None = None
    fill_volume: Decimal | None = None
    error_code: int = 0
    error_message: str = ""


class IExecutionAdapter(ABC):
    """
    Every broker implements this interface.
    MT5, Interactive Brokers, Binance, simulator — all expose the same methods.

    The Kernel interacts only with this interface. It never knows which broker
    is behind it.
    """

    # ── Lifecycle ────────────────────────────────────

    @abstractmethod
    def connect(self) -> bool:
        """Establish connection to the broker. Returns True on success."""
        ...

    @abstractmethod
    def disconnect(self) -> None:
        """Clean up connection resources."""
        ...

    # ── Order submission ─────────────────────────────

    @abstractmethod
    def submit_market_order(
        self,
        symbol: str,
        direction: str,
        volume: Decimal,
        sl: Optional[Decimal] = None,
        tp: Optional[Decimal] = None,
        strategy_id: str = "",
    ) -> AdapterOrderResult:
        ...

    @abstractmethod
    def submit_limit_order(
        self,
        symbol: str,
        direction: str,
        volume: Decimal,
        price: Decimal,
        sl: Optional[Decimal] = None,
        tp: Optional[Decimal] = None,
        strategy_id: str = "",
    ) -> AdapterOrderResult:
        ...

    @abstractmethod
    def submit_stop_order(
        self,
        symbol: str,
        direction: str,
        volume: Decimal,
        price: Decimal,
        sl: Optional[Decimal] = None,
        tp: Optional[Decimal] = None,
        strategy_id: str = "",
    ) -> AdapterOrderResult:
        ...

    # ── Order/Position management ────────────────────

    @abstractmethod
    def cancel_order(self, ticket: int) -> AdapterOrderResult:
        ...

    @abstractmethod
    def close_position(self, ticket: int) -> AdapterOrderResult:
        ...

    @abstractmethod
    def modify_position(
        self,
        ticket: int,
        sl: Optional[Decimal] = None,
        tp: Optional[Decimal] = None,
    ) -> AdapterOrderResult:
        ...

    # ── Account queries ──────────────────────────────

    @abstractmethod
    def get_balance(self) -> Decimal:
        ...

    @abstractmethod
    def get_equity(self) -> Decimal:
        ...

    @abstractmethod
    def get_used_margin(self) -> Decimal:
        ...

    @abstractmethod
    def get_free_margin(self) -> Decimal:
        ...

    # ── Position/Order queries ───────────────────────

    @abstractmethod
    def get_open_positions(self, strategy_id: str = None) -> list[dict]:
        """Return all open positions, optionally filtered by strategy_id."""
        ...

    @abstractmethod
    def get_pending_orders(self, strategy_id: str = None) -> list[dict]:
        """Return all pending orders, optionally filtered by strategy_id."""
        ...

    # ── Symbol info ──────────────────────────────────

    @abstractmethod
    def get_symbol_info(self, symbol: str) -> dict:
        """Return symbol properties (digits, volume_min, point, etc.)."""
        ...
```

### Reference implementation — Data Adapter

```python
class IDataAdapter(ABC):
    """
    Every data source implements this interface.
    CSV files, MT5 terminal, Binance websocket, database — all expose the same methods.
    """

    # ── Lifecycle ────────────────────────────────────

    @abstractmethod
    def connect(self) -> bool:
        ...

    @abstractmethod
    def disconnect(self) -> None:
        ...

    # ── Subscription ─────────────────────────────────

    @abstractmethod
    def subscribe(self, symbol: str, timeframe: str) -> None:
        """Register interest in a symbol/timeframe pair."""
        ...

    # ── Bar data ─────────────────────────────────────

    @abstractmethod
    def get_next_bar(self) -> BarEvent | None:
        """
        Return the next bar in chronological order, or None if no more data.
        For backtest: reads from file/memory. For live: blocks until new bar arrives.
        """
        ...

    @abstractmethod
    def get_latest_bars(self, symbol: str, timeframe: str, count: int):
        """Return the last N bars as a polars DataFrame."""
        ...

    # ── Tick data ────────────────────────────────────

    @abstractmethod
    def get_latest_tick(self, symbol: str) -> dict:
        """Return latest bid/ask/last for a symbol."""
        ...

    @abstractmethod
    def get_latest_bid(self, symbol: str) -> Decimal:
        ...

    @abstractmethod
    def get_latest_ask(self, symbol: str) -> Decimal:
        ...

    # ── State ────────────────────────────────────────

    @property
    @abstractmethod
    def has_more_data(self) -> bool:
        """False when backtest data is exhausted. Always True for live."""
        ...
```

### Concrete adapter examples

#### Simulator (backtest)

```python
class SimulatorExecutionAdapter(IExecutionAdapter):
    """
    In-memory order execution for backtesting.
    No broker connection. Fills at the requested price.
    Equivalent to PyEventBT's Mt5SimulatorExecutionEngineConnector.
    """

    def __init__(self, initial_balance: Decimal = Decimal("10000")):
        self._balance = initial_balance
        self._equity = initial_balance
        self._positions: dict[int, dict] = {}
        self._pending_orders: dict[int, dict] = {}
        self._next_ticket = 1
        self._margin = Decimal("0")

    def connect(self) -> bool:
        return True

    def disconnect(self) -> None:
        pass

    def submit_market_order(self, symbol, direction, volume,
                            sl=None, tp=None, strategy_id=""):
        ticket = self._next_ticket
        self._next_ticket += 1
        # In a full implementation: calculate margin, check available margin,
        # determine fill price from latest bar, update balance
        self._positions[ticket] = {
            "ticket": ticket, "symbol": symbol, "direction": direction,
            "volume": volume, "sl": sl, "tp": tp, "strategy_id": strategy_id,
        }
        return AdapterOrderResult(
            success=True,
            order_ticket=ticket,
            fill_price=Decimal("0"),  # filled from current bar price
            fill_volume=volume,
        )

    def close_position(self, ticket):
        pos = self._positions.pop(ticket, None)
        if pos is None:
            return AdapterOrderResult(success=False, error_message="Position not found")
        return AdapterOrderResult(success=True, order_ticket=ticket)

    def get_balance(self) -> Decimal:
        return self._balance

    def get_equity(self) -> Decimal:
        return self._equity

    # ... remaining methods follow the same pattern
```

#### CSV data (backtest)

```python
class CSVDataAdapter(IDataAdapter):
    """
    Reads bar data from CSV files.
    Equivalent to PyEventBT's CSVBacktestDataConfig path.
    """

    def __init__(self, csv_configs: list[dict]):
        """
        csv_configs: [{"symbol": "EURUSD", "timeframe": "1h", "path": "data/eurusd_1h.csv"}]
        """
        self._configs = csv_configs
        self._dataframes = {}     # "EURUSD_1h" → polars DataFrame
        self._cursors = {}        # "EURUSD_1h" → current row index
        self._subscriptions = []  # list of (symbol, timeframe) pairs

    def connect(self) -> bool:
        import polars as pl
        for cfg in self._configs:
            key = f"{cfg['symbol']}_{cfg['timeframe']}"
            self._dataframes[key] = pl.read_csv(cfg["path"])
            self._cursors[key] = 0
        return True

    def disconnect(self) -> None:
        self._dataframes.clear()
        self._cursors.clear()

    def subscribe(self, symbol: str, timeframe: str) -> None:
        self._subscriptions.append((symbol, timeframe))

    def get_next_bar(self) -> BarEvent | None:
        # Find the subscription with the earliest next bar
        earliest_key = None
        earliest_time = None
        for symbol, tf in self._subscriptions:
            key = f"{symbol}_{tf}"
            if self._cursors.get(key, 0) >= len(self._dataframes.get(key, [])):
                continue
            row = self._dataframes[key].row(self._cursors[key], named=True)
            bar_time = row["time"]  # depends on CSV format
            if earliest_time is None or bar_time < earliest_time:
                earliest_time = bar_time
                earliest_key = key
        if earliest_key is None:
            return None
        row = self._dataframes[earliest_key].row(self._cursors[earliest_key], named=True)
        self._cursors[earliest_key] += 1
        # Convert row to BarEvent (implementation depends on CSV column format)
        return self._row_to_bar_event(earliest_key, row)

    def get_latest_bars(self, symbol, timeframe, count):
        key = f"{symbol}_{timeframe}"
        cursor = self._cursors.get(key, 0)
        start = max(0, cursor - count)
        return self._dataframes[key].slice(start, cursor - start)

    @property
    def has_more_data(self) -> bool:
        return any(
            self._cursors.get(f"{s}_{tf}", 0) < len(self._dataframes.get(f"{s}_{tf}", []))
            for s, tf in self._subscriptions
        )
```

#### MT5 live execution

```python
class MT5ExecutionAdapter(IExecutionAdapter):
    """
    Live order execution via MetaTrader 5 Python package.
    Wraps the mt5 module behind the standard IExecutionAdapter interface.
    """

    def __init__(self, account: int, password: str, server: str):
        self._account = account
        self._password = password
        self._server = server

    def connect(self) -> bool:
        import MetaTrader5 as mt5
        if not mt5.initialize():
            return False
        return mt5.login(self._account, password=self._password, server=self._server)

    def disconnect(self) -> None:
        import MetaTrader5 as mt5
        mt5.shutdown()

    def submit_market_order(self, symbol, direction, volume,
                            sl=None, tp=None, strategy_id=""):
        import MetaTrader5 as mt5
        order_type = mt5.ORDER_TYPE_BUY if direction == "BUY" else mt5.ORDER_TYPE_SELL
        tick = mt5.symbol_info_tick(symbol)
        price = tick.ask if direction == "BUY" else tick.bid

        request = {
            "action": mt5.TRADE_ACTION_DEAL,
            "symbol": symbol,
            "volume": float(volume),
            "type": order_type,
            "price": price,
            "magic": int(strategy_id) if strategy_id else 0,
            "comment": f"strategy_{strategy_id}",
            "type_time": mt5.ORDER_TIME_GTC,
            "type_filling": mt5.ORDER_FILLING_IOC,
        }
        if sl:
            request["sl"] = float(sl)
        if tp:
            request["tp"] = float(tp)

        result = mt5.order_send(request)
        return AdapterOrderResult(
            success=(result.retcode == mt5.TRADE_RETCODE_DONE),
            order_ticket=result.order if result.retcode == mt5.TRADE_RETCODE_DONE else None,
            fill_price=Decimal(str(result.price)) if result.price else None,
            fill_volume=Decimal(str(result.volume)) if result.volume else None,
            error_code=result.retcode,
            error_message=result.comment if result.retcode != mt5.TRADE_RETCODE_DONE else "",
        )

    def get_balance(self) -> Decimal:
        import MetaTrader5 as mt5
        info = mt5.account_info()
        return Decimal(str(info.balance))

    def get_equity(self) -> Decimal:
        import MetaTrader5 as mt5
        info = mt5.account_info()
        return Decimal(str(info.equity))

    # ... remaining methods wrap mt5 calls the same way
```

### Routing adapter for multi-broker

```python
class RoutingExecutionAdapter(IExecutionAdapter):
    """
    Routes orders to the correct broker adapter based on symbol patterns.

    Example:
        router = RoutingExecutionAdapter()
        router.add_route("EUR", mt5_adapter)   # EURUSD, EURGBP → MT5
        router.add_route("AAPL", ibkr_adapter) # US stocks → Interactive Brokers
        router.set_default(mt5_adapter)         # everything else → MT5
    """

    def __init__(self):
        self._routes: list[tuple[str, IExecutionAdapter]] = []
        self._default: IExecutionAdapter | None = None

    def add_route(self, symbol_prefix: str, adapter: IExecutionAdapter):
        self._routes.append((symbol_prefix, adapter))

    def set_default(self, adapter: IExecutionAdapter):
        self._default = adapter

    def _resolve(self, symbol: str) -> IExecutionAdapter:
        for prefix, adapter in self._routes:
            if symbol.startswith(prefix):
                return adapter
        if self._default:
            return self._default
        raise ValueError(f"No adapter configured for symbol: {symbol}")

    def connect(self) -> bool:
        adapters = set(a for _, a in self._routes)
        if self._default:
            adapters.add(self._default)
        return all(a.connect() for a in adapters)

    def disconnect(self) -> None:
        adapters = set(a for _, a in self._routes)
        if self._default:
            adapters.add(self._default)
        for a in adapters:
            a.disconnect()

    def submit_market_order(self, symbol, direction, volume,
                            sl=None, tp=None, strategy_id=""):
        return self._resolve(symbol).submit_market_order(
            symbol, direction, volume, sl, tp, strategy_id)

    def close_position(self, ticket):
        # For close/cancel: need to know which adapter owns the ticket.
        # Option 1: Try all adapters. Option 2: Keep a ticket→adapter map.
        # Here we use option 2:
        return self._ticket_map[ticket].close_position(ticket)

    # ... all other methods delegate to _resolve(symbol) or _ticket_map[ticket]
```

### Multi-account adapter

```python
class MultiAccountExecutionAdapter(IExecutionAdapter):
    """
    Routes orders to the correct account within the same broker.

    Example:
        multi = MultiAccountExecutionAdapter()
        multi.add_account("aggressive", MT5ExecutionAdapter(account=111, ...))
        multi.add_account("conservative", MT5ExecutionAdapter(account=222, ...))

    Strategy specifies target account via strategy_id mapping.
    """

    def __init__(self):
        self._accounts: dict[str, IExecutionAdapter] = {}
        self._strategy_to_account: dict[str, str] = {}

    def add_account(self, account_name: str, adapter: IExecutionAdapter):
        self._accounts[account_name] = adapter

    def map_strategy(self, strategy_id: str, account_name: str):
        self._strategy_to_account[strategy_id] = account_name

    def _resolve(self, strategy_id: str) -> IExecutionAdapter:
        account_name = self._strategy_to_account.get(strategy_id)
        if account_name is None:
            raise ValueError(f"No account mapped for strategy: {strategy_id}")
        return self._accounts[account_name]

    def submit_market_order(self, symbol, direction, volume,
                            sl=None, tp=None, strategy_id=""):
        return self._resolve(strategy_id).submit_market_order(
            symbol, direction, volume, sl, tp, strategy_id)

    # ... all other methods delegate to _resolve(strategy_id)
```

---

## 33.4 Event Journal — Crash Recovery via Event Sourcing

### Problem it solves

PyEventBT keeps all state in memory. If the process crashes during live trading, open position tracking, account balance, and pending orders are lost. There is no way to recover or replay.

### Contract

```python
class IEventJournal(Protocol):
    """Append-only event log."""

    def record(self, topic: str, event) -> int:
        """Persist an event. Returns sequence number."""

    def replay(self, from_seq: int = 0) -> Iterator[dict]:
        """Yield journal entries starting from a sequence number."""

    def close(self) -> None:
        """Flush and close the journal file."""


class ISnapshotManager(Protocol):
    """Periodic Cache state serialization."""

    def save(self, cache: ICache, journal_seq: int) -> None:
        """Serialize full Cache state at a point in the journal."""

    def load_latest(self) -> tuple[dict | None, int]:
        """Load most recent snapshot. Returns (data, journal_seq)."""

    def restore_cache(self, cache: Cache, snapshot: dict) -> None:
        """Apply a snapshot to a Cache instance."""
```

### Behavior

| Rule | Description |
|---|---|
| **Append-only** | Events are written sequentially, never modified or deleted. |
| **Journal before dispatch** | An event is persisted to disk BEFORE it is processed by handlers. If the process crashes mid-processing, the event can be replayed on recovery. |
| **Monotonic sequence** | Every entry gets a strictly increasing sequence number. Used for replay positioning. |
| **Durable writes** | `record()` must `flush()` and `fsync()` to ensure the event survives a crash. |
| **Snapshot interval** | Snapshots are taken every N bars (configurable). This bounds recovery time — replay only events since last snapshot. |
| **Recovery order** | On startup: load latest snapshot → restore Cache → replay journal entries after snapshot's sequence number. |
| **Deterministic replay** | Replaying the same journal from the same snapshot must produce identical Cache state. |

### Protocol

```
Normal operation:
    Kernel ──record("event.bar", bar)──▶ Journal (to disk)
    Kernel ──publish("event.bar", bar)──▶ Bus (to handlers)
    ...every N bars...
    Kernel ──save(cache, seq)──▶ SnapshotManager (to disk)

Recovery (on startup):
    SnapshotManager ──load_latest()──▶ Kernel
    Kernel ──restore_cache(cache, snapshot)──▶ Cache
    Journal ──replay(from_seq=snapshot_seq)──▶ Kernel
    Kernel ──(re-dispatches each event)──▶ Bus → handlers → Cache

The Journal is write-only during normal operation, read-only during recovery.
```

### Reference implementation

```python
import json
import os
from datetime import datetime
from decimal import Decimal
from pathlib import Path


class EventJournal:
    """
    Append-only event log for crash recovery and audit.

    Usage:
        journal = EventJournal("./journal/events.jsonl")

        # Record every input event before processing
        journal.record("event.bar", bar_event)
        journal.record("event.fill", fill_event)

        # On crash recovery: replay from last snapshot
        for entry in journal.replay(from_seq=last_snapshot_seq):
            kernel.process(entry)

        # Periodic snapshots to avoid full replay
        journal.snapshot(cache)
    """

    def __init__(self, journal_path: str):
        Path(journal_path).parent.mkdir(parents=True, exist_ok=True)
        self._path = journal_path
        self._file = open(journal_path, "a")
        self._sequence = self._count_existing_entries()

    def _count_existing_entries(self) -> int:
        if not os.path.exists(self._path):
            return 0
        with open(self._path, "r") as f:
            return sum(1 for _ in f)

    def record(self, topic: str, event) -> int:
        """
        Persist an event to the journal. Returns sequence number.
        Call this BEFORE dispatching the event to handlers.
        """
        self._sequence += 1
        entry = {
            "seq": self._sequence,
            "ts": datetime.now().isoformat(),
            "topic": topic,
            "type": event.__class__.__name__,
            "data": self._serialize_event(event),
        }
        self._file.write(json.dumps(entry, default=str) + "\n")
        self._file.flush()
        os.fsync(self._file.fileno())   # ensure durability
        return self._sequence

    def replay(self, from_seq: int = 0):
        """Yield journal entries starting from a sequence number."""
        with open(self._path, "r") as f:
            for line in f:
                if not line.strip():
                    continue
                entry = json.loads(line)
                if entry["seq"] > from_seq:
                    yield entry

    def close(self):
        self._file.close()

    @staticmethod
    def _serialize_event(event) -> dict:
        result = {}
        for key, value in event.__dict__.items():
            if isinstance(value, Decimal):
                result[key] = {"_type": "Decimal", "value": str(value)}
            elif isinstance(value, datetime):
                result[key] = {"_type": "datetime", "value": value.isoformat()}
            elif hasattr(value, "value") and hasattr(value, "name"):  # Enum
                result[key] = {"_type": "Enum", "enum_class": type(value).__name__,
                               "value": value.value}
            elif hasattr(value, "__dict__"):  # nested dataclass
                result[key] = {"_type": value.__class__.__name__,
                               **EventJournal._serialize_event(value)}
            else:
                result[key] = value
        return result


class SnapshotManager:
    """
    Periodic Cache snapshots to avoid replaying the full journal on recovery.

    Recovery process:
    1. Load the latest snapshot → restore Cache state
    2. Replay journal entries AFTER the snapshot's sequence number
    """

    def __init__(self, snapshot_dir: str):
        self._dir = snapshot_dir
        Path(snapshot_dir).mkdir(parents=True, exist_ok=True)

    def save(self, cache: Cache, journal_seq: int):
        """Serialize the full Cache state at a point in time."""
        snapshot = {
            "journal_seq": journal_seq,
            "ts": datetime.now().isoformat(),
            "account": {
                "balance": str(cache.account.balance),
                "equity": str(cache.account.equity),
                "margin": str(cache.account.margin),
                "margin_free": str(cache.account.margin_free),
                "currency": cache.account.currency,
            },
            "positions": [
                {
                    "ticket": p.ticket,
                    "symbol": p.symbol,
                    "direction": p.direction,
                    "volume": str(p.volume),
                    "price_entry": str(p.price_entry),
                    "strategy_id": p.strategy_id,
                    "sl": str(p.sl) if p.sl else None,
                    "tp": str(p.tp) if p.tp else None,
                }
                for p in cache.get_positions()
            ],
            "symbols": {
                sym: {
                    "digits": info.digits,
                    "volume_min": str(info.volume_min),
                    "volume_max": str(info.volume_max),
                    "point": str(info.point),
                }
                for sym, info in cache._symbols.items()
            },
        }
        path = os.path.join(self._dir, f"snapshot_{journal_seq}.json")
        with open(path, "w") as f:
            json.dump(snapshot, f, indent=2)

    def load_latest(self) -> tuple[dict | None, int]:
        """Load the most recent snapshot. Returns (snapshot_data, journal_seq)."""
        snapshots = sorted(Path(self._dir).glob("snapshot_*.json"))
        if not snapshots:
            return None, 0
        with open(snapshots[-1], "r") as f:
            data = json.load(f)
        return data, data["journal_seq"]

    def restore_cache(self, cache: Cache, snapshot: dict):
        """Apply a snapshot to a Cache instance."""
        acc = snapshot["account"]
        cache.update_account(
            balance=Decimal(acc["balance"]),
            equity=Decimal(acc["equity"]),
            margin=Decimal(acc["margin"]),
            margin_free=Decimal(acc["margin_free"]),
        )
        for pos in snapshot["positions"]:
            cache.update_position(PositionSnapshot(
                ticket=pos["ticket"],
                symbol=pos["symbol"],
                direction=pos["direction"],
                volume=Decimal(pos["volume"]),
                price_entry=Decimal(pos["price_entry"]),
                unrealized_pnl=Decimal("0"),
                strategy_id=pos["strategy_id"],
                sl=Decimal(pos["sl"]) if pos["sl"] else None,
                tp=Decimal(pos["tp"]) if pos["tp"] else None,
            ))
        for sym, info in snapshot["symbols"].items():
            cache.set_symbol_info(sym, SymbolState(
                digits=info["digits"],
                volume_min=Decimal(info["volume_min"]),
                volume_max=Decimal(info["volume_max"]),
                point=Decimal(info["point"]),
            ))
```

### How it plugs in — the Kernel uses journal + snapshots

```python
class Kernel:
    def __init__(self, ...):
        self.journal = EventJournal("./journal/events.jsonl")
        self.snapshots = SnapshotManager("./journal/snapshots/")
        self._bars_since_snapshot = 0
        self.SNAPSHOT_INTERVAL = 1000  # snapshot every N bars

    def run(self, data_adapter):
        # ── Recovery: load last snapshot, replay remaining events ──
        snapshot, last_seq = self.snapshots.load_latest()
        if snapshot:
            self.snapshots.restore_cache(self.cache, snapshot)
            for entry in self.journal.replay(from_seq=last_seq):
                self._replay_entry(entry)

        # ── Normal operation ──
        while data_adapter.has_more_data or not self.bus.is_empty:
            if self.bus.is_empty:
                bar = data_adapter.get_next_bar()
                if bar:
                    self.journal.record("event.bar", bar)  # journal BEFORE dispatch
                    self.bus.publish("event.bar", bar)

                    self._bars_since_snapshot += 1
                    if self._bars_since_snapshot >= self.SNAPSHOT_INTERVAL:
                        self.snapshots.save(self.cache, self.journal._sequence)
                        self._bars_since_snapshot = 0
            else:
                self.bus.dispatch_next()
```

---

## 33.5 Kernel — Full Event Loop Assembling All Components

### What it replaces

PyEventBT's `TradingDirector` + `PortfolioHandler` + direct wiring in `Strategy.backtest()`.

### Contract

```python
class IKernel(Protocol):
    """Central orchestrator. Owns all infrastructure."""

    # ── Infrastructure (owned by Kernel) ──
    bus: IMessageBus
    cache: ICache          # Kernel is the single writer
    journal: IEventJournal  # optional, for live trading

    # ── Lifecycle ──
    def run(self) -> None:
        """Start the event loop. Blocks until data is exhausted (backtest) or stopped (live)."""

    def stop(self) -> None:
        """Signal the loop to stop after processing the current event."""
```

### Behavior

| Rule | Description |
|---|---|
| **Single-threaded** | The entire `run()` loop executes on one thread. No concurrency, no locks, no race conditions. |
| **Single writer** | Only the Kernel writes to the Cache. After processing a FillEvent, Kernel updates positions and account state in Cache before dispatching the next event. |
| **Journal before dispatch** | In live mode, every input event is journaled before being published to the bus. |
| **Event ordering** | One event fully processed (including all events it generates) before the next is dequeued from the data adapter. |
| **BAR processing order** | On each bar: 1) Check SL/TP hits → 2) Run scheduled callbacks → 3) Generate signals. This order is invariant. |
| **Signal chain** | SIGNAL → Sizing → Risk → ORDER is synchronous within one dispatch cycle. Not three separate dispatches. |
| **Command handling** | Commands published by user code (close positions, cancel orders) are queued and processed on the next dispatch cycle. |
| **Adapter isolation** | The Kernel is the only component that calls adapter methods. No other component touches adapters. |
| **Snapshot interval** | In live mode, the Kernel takes a Cache snapshot every N bars (configurable). |

### Protocol

```
┌─────────────────────────────────────────────────────────────────────┐
│                         KERNEL EVENT LOOP                           │
│                                                                     │
│  while running:                                                     │
│      if bus.is_empty:                                               │
│          bar = data_adapter.get_next_bar()                          │
│          if bar:                                                    │
│              journal.record("event.bar", bar)    # persist first    │
│              bus.publish("event.bar", bar)        # then dispatch   │
│      else:                                                          │
│          bus.dispatch_next()                      # process one     │
│                                                                     │
│  Event routing (via bus subscriptions):                              │
│      "event.bar"    → Kernel._on_bar()                              │
│      "event.signal" → Kernel._on_signal()                           │
│      "event.order"  → Kernel._on_order()                            │
│      "event.fill"   → Kernel._on_fill()     → updates Cache        │
│      "command.*"    → Kernel._on_*_command() → calls adapter        │
│                                                                     │
│  Request routing (via bus request handlers):                         │
│      "data.latest_bars" → data_adapter.get_latest_bars()            │
│      "data.latest_tick" → data_adapter.get_latest_tick()            │
│      "data.latest_bid"  → data_adapter.get_latest_bid()             │
│      "data.latest_ask"  → data_adapter.get_latest_ask()             │
└─────────────────────────────────────────────────────────────────────┘
```

### Reference implementation

```python
class Kernel:
    """
    Central orchestrator. Replaces TradingDirector.

    Owns:
    - MessageBus (event routing)
    - Cache (single-writer state)
    - EventJournal (crash recovery)
    - Adapters (broker/data abstraction)

    Single-threaded. All dispatch happens in the run() loop.
    """

    def __init__(
        self,
        data_adapter: IDataAdapter,
        exec_adapter: IExecutionAdapter,
        signal_engine,
        sizing_engine,
        risk_engine,
        initial_balance: Decimal = Decimal("10000"),
        journal_path: str = None,
    ):
        # ── Core infrastructure ──
        self.bus = MessageBus()
        self.cache = Cache()
        self.journal = EventJournal(journal_path) if journal_path else None

        # ── Adapters ──
        self.data_adapter = data_adapter
        self.exec_adapter = exec_adapter

        # ── Engines ──
        self.signal_engine = signal_engine
        self.sizing_engine = sizing_engine
        self.risk_engine = risk_engine

        # ── Initialize cache ──
        self.cache.update_account(
            balance=initial_balance,
            equity=initial_balance,
            margin=Decimal("0"),
            margin_free=initial_balance,
        )

        # ── Wire subscriptions ──
        self.bus.subscribe("event.bar", self._on_bar)
        self.bus.subscribe("event.signal", self._on_signal)
        self.bus.subscribe("event.order", self._on_order)
        self.bus.subscribe("event.fill", self._on_fill)
        self.bus.subscribe("command.close_position", self._on_close_position)
        self.bus.subscribe("command.close_positions", self._on_close_positions)
        self.bus.subscribe("command.cancel_order", self._on_cancel_order)
        self.bus.subscribe("command.modify_position", self._on_modify_position)

        # ── Request handlers ──
        self.bus.register_request_handler(
            "data.latest_bars",
            lambda symbol, timeframe, count:
                self.data_adapter.get_latest_bars(symbol, timeframe, count))
        self.bus.register_request_handler(
            "data.latest_tick",
            lambda symbol: self.data_adapter.get_latest_tick(symbol))
        self.bus.register_request_handler(
            "data.latest_bid",
            lambda symbol: self.data_adapter.get_latest_bid(symbol))
        self.bus.register_request_handler(
            "data.latest_ask",
            lambda symbol: self.data_adapter.get_latest_ask(symbol))

    def run(self):
        """Main event loop."""
        self.data_adapter.connect()
        self.exec_adapter.connect()

        while self.data_adapter.has_more_data or not self.bus.is_empty:
            if self.bus.is_empty:
                bar = self.data_adapter.get_next_bar()
                if bar:
                    if self.journal:
                        self.journal.record("event.bar", bar)
                    self.bus.publish("event.bar", bar)
            else:
                self.bus.dispatch_next()

        self.data_adapter.disconnect()
        self.exec_adapter.disconnect()
        if self.journal:
            self.journal.close()

    # ── Event handlers ───────────────────────────────

    def _on_bar(self, bar: BarEvent):
        # 1. Check SL/TP on open positions
        self._check_sl_tp(bar)

        # 2. Generate signal
        ctx = StrategyContext(
            bus=self.bus, cache=self.cache,
            strategy_id=self.signal_engine.strategy_id,
            trading_context="BACKTEST",
        )
        result = self.signal_engine.generate_signal(bar, ctx)
        if result:
            signals = result if isinstance(result, list) else [result]
            for signal in signals:
                if self.journal:
                    self.journal.record("event.signal", signal)
                self.bus.publish("event.signal", signal)

    def _on_signal(self, signal: SignalEvent):
        # Sizing
        suggested = self.sizing_engine.get_suggested_order(signal, self.cache)
        if not suggested or suggested.volume <= 0:
            return

        # Risk
        approved = self.risk_engine.assess_order(suggested, self.cache)
        if not approved:
            return

        # Emit order
        order = OrderEvent(
            symbol=signal.symbol,
            time_generated=signal.time_generated,
            strategy_id=signal.strategy_id,
            volume=suggested.volume,
            signal_type=signal.signal_type,
            order_type=signal.order_type,
            order_price=signal.order_price,
            sl=signal.sl,
            tp=signal.tp,
            buffer_data=suggested.buffer_data,
        )
        if self.journal:
            self.journal.record("event.order", order)
        self.bus.publish("event.order", order)

    def _on_order(self, order: OrderEvent):
        # Route to adapter
        if order.order_type == OrderType.MARKET:
            result = self.exec_adapter.submit_market_order(
                symbol=order.symbol,
                direction=order.signal_type.value,
                volume=order.volume,
                sl=order.sl,
                tp=order.tp,
                strategy_id=order.strategy_id,
            )
        elif order.order_type == OrderType.LIMIT:
            result = self.exec_adapter.submit_limit_order(
                symbol=order.symbol,
                direction=order.signal_type.value,
                volume=order.volume,
                price=order.order_price,
                sl=order.sl,
                tp=order.tp,
                strategy_id=order.strategy_id,
            )
        else:
            result = self.exec_adapter.submit_stop_order(
                symbol=order.symbol,
                direction=order.signal_type.value,
                volume=order.volume,
                price=order.order_price,
                sl=order.sl,
                tp=order.tp,
                strategy_id=order.strategy_id,
            )

        if result.success:
            fill = FillEvent(
                deal=DealType.IN,
                symbol=order.symbol,
                time_generated=order.time_generated,
                position_id=result.order_ticket,
                strategy_id=order.strategy_id,
                volume=result.fill_volume,
                price=result.fill_price,
                signal_type=order.signal_type,
                commission=Decimal("0"),
                swap=Decimal("0"),
                fee=Decimal("0"),
                gross_profit=Decimal("0"),
                ccy=self.cache.account.currency,
                exchange="",
            )
            if self.journal:
                self.journal.record("event.fill", fill)
            self.bus.publish("event.fill", fill)

    def _on_fill(self, fill: FillEvent):
        # Single writer: only Kernel updates the Cache
        if fill.deal == DealType.IN:
            self.cache.update_position(PositionSnapshot(
                ticket=fill.position_id,
                symbol=fill.symbol,
                direction=fill.signal_type.value,
                volume=fill.volume,
                price_entry=fill.price,
                unrealized_pnl=Decimal("0"),
                strategy_id=fill.strategy_id,
            ))
        elif fill.deal == DealType.OUT:
            self.cache.remove_position(fill.position_id)

        self.cache.update_account(
            balance=self.exec_adapter.get_balance(),
            equity=self.exec_adapter.get_equity(),
            margin=self.exec_adapter.get_used_margin(),
            margin_free=self.exec_adapter.get_free_margin(),
        )

    # ── Command handlers ─────────────────────────────

    def _on_close_position(self, cmd: dict):
        result = self.exec_adapter.close_position(cmd["ticket"])
        if result.success:
            # Emit a FillEvent for the close
            pos = self.cache._positions.get(cmd["ticket"])
            if pos:
                fill = FillEvent(
                    deal=DealType.OUT,
                    symbol=pos.symbol,
                    position_id=cmd["ticket"],
                    strategy_id=cmd["strategy_id"],
                    volume=pos.volume,
                    price=result.fill_price or Decimal("0"),
                    signal_type=pos.direction,
                    # ...
                )
                self.bus.publish("event.fill", fill)

    def _on_close_positions(self, cmd: dict):
        positions = self.cache.get_positions(
            strategy_id=cmd.get("strategy_id"),
            symbol=cmd.get("symbol"),
        )
        if cmd.get("direction"):
            positions = [p for p in positions if p.direction == cmd["direction"]]
        for pos in positions:
            self._on_close_position({"ticket": pos.ticket, "strategy_id": pos.strategy_id})

    def _on_cancel_order(self, cmd: dict):
        result = self.exec_adapter.cancel_order(cmd["ticket"])
        if result.success:
            self.cache.remove_pending_order(cmd["ticket"])

    def _on_modify_position(self, cmd: dict):
        self.exec_adapter.modify_position(
            ticket=cmd["ticket"],
            sl=cmd.get("sl"),
            tp=cmd.get("tp"),
        )

    # ── Internal helpers ─────────────────────────────

    def _check_sl_tp(self, bar: BarEvent):
        """Check if any open position's SL or TP was hit by this bar."""
        for pos in self.cache.get_positions(symbol=bar.symbol):
            hit = False
            if pos.direction == "BUY":
                if pos.sl and bar.data.low_f <= float(pos.sl):
                    hit = True
                elif pos.tp and bar.data.high_f >= float(pos.tp):
                    hit = True
            elif pos.direction == "SELL":
                if pos.sl and bar.data.high_f >= float(pos.sl):
                    hit = True
                elif pos.tp and bar.data.low_f <= float(pos.tp):
                    hit = True
            if hit:
                self._on_close_position({
                    "ticket": pos.ticket,
                    "strategy_id": pos.strategy_id,
                })
```

### Usage — how a user sets up and runs a backtest

```python
from my_framework import (
    Kernel, CSVDataAdapter, SimulatorExecutionAdapter,
    StrategyContext, BarEvent, SignalEvent, SignalType, OrderType,
)
from decimal import Decimal

# ── 1. Define strategy logic (same as PyEventBT) ────

def my_signal_engine(bar: BarEvent, ctx: StrategyContext):
    bars = ctx.get_latest_bars(bar.symbol, bar.timeframe, 50)
    closes = bars.select("close").to_numpy().flatten()

    sma_fast = closes[-10:].mean()
    sma_slow = closes[-50:].mean()

    open_positions = ctx.get_position_count(symbol=bar.symbol)

    if sma_fast > sma_slow and open_positions == 0:
        return SignalEvent(
            symbol=bar.symbol,
            signal_type=SignalType.BUY,
            order_type=OrderType.MARKET,
            sl=Decimal(str(closes[-1] * 0.98)),
            tp=Decimal(str(closes[-1] * 1.04)),
            # ...
        )
    return None

my_signal_engine.strategy_id = "1001"


# ── 2. Wire components ──────────────────────────────

data_adapter = CSVDataAdapter([
    {"symbol": "EURUSD", "timeframe": "1h", "path": "data/eurusd_1h.csv"},
])

exec_adapter = SimulatorExecutionAdapter(initial_balance=Decimal("10000"))

kernel = Kernel(
    data_adapter=data_adapter,
    exec_adapter=exec_adapter,
    signal_engine=my_signal_engine,
    sizing_engine=FixedSizingEngine(volume=Decimal("0.1")),
    risk_engine=PassthroughRiskEngine(),
    initial_balance=Decimal("10000"),
    journal_path="./journal/backtest_001.jsonl",
)


# ── 3. Run ───────────────────────────────────────────

kernel.run()
```

---

## 33.6 Component Map — What Replaces What

| PyEventBT component | New component | Key difference |
|---|---|---|
| `SharedData` (class-level singleton) | `Cache` (instance, single-writer) | Only Kernel writes; components read |
| `Modules` (live object references) | `StrategyContext` (bus + cache wrapper) | No direct access to other components |
| `queue.Queue` (simple FIFO) | `MessageBus` (pub/sub + req/resp) | Multiple subscribers, request/response |
| `TradingDirector` (event loop) | `Kernel` (event loop + cache + journal) | Owns all infrastructure, single writer |
| `PortfolioHandler` (sync chain orchestrator) | `Kernel._on_signal` (bus-based chain) | Sizing→Risk→Order through bus, not sync calls |
| `Mt5SimulatorExecutionEngineConnector` | `SimulatorExecutionAdapter` (implements `IExecutionAdapter`) | Formal interface, standardized results |
| `Mt5LiveExecutionEngineConnector` | `MT5ExecutionAdapter` (implements `IExecutionAdapter`) | Same interface as simulator |
| CSV loading in DataProvider | `CSVDataAdapter` (implements `IDataAdapter`) | Same interface as live data |
| *(nothing)* | `EventJournal` + `SnapshotManager` | Crash recovery, audit, replay |
| *(nothing)* | `RoutingExecutionAdapter` | Multi-broker order routing |
| *(nothing)* | `MultiAccountExecutionAdapter` | Multi-account order routing |

---
