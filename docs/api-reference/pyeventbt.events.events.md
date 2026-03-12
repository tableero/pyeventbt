# File: `pyeventbt/events/events.py`

## Module

`pyeventbt.events.events`

## Purpose

Defines every event type that transits the shared event queue, plus the lightweight `Bar` data payload. This is the canonical schema for all inter-component communication in PyEventBT.

## Tags

`events`, `core`, `data-model`, `pydantic`, `dataclass`, `enum`, `bar-data`

## Dependencies

| Dependency | Import |
|---|---|
| `pydantic` | `BaseModel`, `Field` |
| `datetime` | `datetime` |
| `typing` | `Optional` |
| `decimal` | `Decimal` |
| `pandas` | `pd` (for `pd.Timestamp`) |
| `enum` | `Enum` |
| `dataclasses` | `dataclass`, `field` |
| `pyeventbt.strategy.core.strategy_timeframes` | `StrategyTimeframes` |

## Classes/Functions

### `EventType(str, Enum)`

Enumeration of all event types flowing through the queue.

| Member | Value | Description |
|---|---|---|
| `BAR` | `"BAR"` | A new price bar has arrived |
| `SIGNAL` | `"SIGNAL"` | A trading signal has been generated |
| `ORDER` | `"ORDER"` | An order is ready for execution |
| `FILL` | `"FILL"` | An order has been filled |
| `SCHEDULED_EVENT` | `"SCHEDULED_EVENT"` | A time-based scheduled callback trigger |

---

### `SignalType(str, Enum)`

Direction of a trading signal.

| Member | Value |
|---|---|
| `BUY` | `"BUY"` |
| `SELL` | `"SELL"` |

---

### `OrderType(str, Enum)`

Type of order to place.

| Member | Value | Description |
|---|---|---|
| `MARKET` | `"MARKET"` | Execute at current market price |
| `LIMIT` | `"LIMIT"` | Execute at specified price or better |
| `STOP` | `"STOP"` | Execute when price reaches a stop level |
| `CONT` | `"CONT"` | Continuation order (rollover semantics) |

---

### `DealType(str, Enum)`

Whether a fill opens or closes a position.

| Member | Value |
|---|---|
| `IN` | `"IN"` (position entry) |
| `OUT` | `"OUT"` (position exit) |

---

### `EventBase(BaseModel)`

```python
class EventBase(BaseModel):
    type: EventType
```

**Description**: Abstract base for all events. Provides the `type` discriminator field and enables `arbitrary_types_allowed` via Pydantic `Config`.

**Attributes**:

| Attribute | Type | Description |
|---|---|---|
| `type` | `EventType` | Discriminator identifying the event kind |

---

### `Bar`

```python
@dataclass(slots=True)
class Bar:
    open: int
    high: int
    low: int
    close: int
    tickvol: int
    volume: int
    spread: int
    digits: int
```

**Description**: Compact price bar payload. All prices are stored as integers; divide by `10 ** digits` to recover float values. Uses `__slots__` for memory efficiency (~56 bytes per instance).

**Attributes**:

| Attribute | Type | Description |
|---|---|---|
| `open` | `int` | Open price as integer |
| `high` | `int` | High price as integer |
| `low` | `int` | Low price as integer |
| `close` | `int` | Close price as integer |
| `tickvol` | `int` | Tick volume |
| `volume` | `int` | Real volume |
| `spread` | `int` | Spread as integer |
| `digits` | `int` | Number of decimal places for price reconstruction |

**Properties**:

| Property | Returns | Description |
|---|---|---|
| `price_factor` | `float` | `10 ** self.digits`. Cached in private `__price_factor` field on first access |
| `open_f` | `float` | `self.open / self.price_factor` |
| `high_f` | `float` | `self.high / self.price_factor` |
| `low_f` | `float` | `self.low / self.price_factor` |
| `close_f` | `float` | `self.close / self.price_factor` |
| `spread_f` | `float` | `self.spread / self.price_factor` |

---

### `BarEvent(EventBase)`

```python
class BarEvent(EventBase):
    type: EventType = EventType.BAR
    symbol: str
    datetime: datetime
    data: Bar
    timeframe: str
```

**Description**: Envelope carrying a `Bar` payload with metadata. Emitted by `DataProvider` when a new bar is available.

**Attributes**:

| Attribute | Type | Default | Description |
|---|---|---|---|
| `type` | `EventType` | `EventType.BAR` | Event discriminator |
| `symbol` | `str` | -- | Trading instrument symbol (e.g., `"EURUSD"`) |
| `datetime` | `datetime` | -- | Bar timestamp |
| `data` | `Bar` | -- | The price bar payload |
| `timeframe` | `str` | -- | Bar timeframe (e.g., `"M1"`, `"H1"`) |

---

### `SignalEvent(EventBase)`

```python
class SignalEvent(EventBase):
    type: EventType = EventType.SIGNAL
    symbol: str
    time_generated: datetime
    strategy_id: str
    forecast: Optional[float] = 0.0
    signal_type: SignalType
    order_type: OrderType
    order_price: Optional[Decimal] = Decimal('0.0')
    sl: Optional[Decimal] = Decimal('0.0')
    tp: Optional[Decimal] = Decimal('0.0')
    rollover: Optional[tuple] = (False, "", "")
```

**Description**: Emitted by `SignalEngineService` when a user-defined signal engine detects a trading opportunity. Carries signal direction, order type, and optional price levels.

**Attributes**:

| Attribute | Type | Default | Description |
|---|---|---|---|
| `type` | `EventType` | `EventType.SIGNAL` | Event discriminator |
| `symbol` | `str` | -- | Trading instrument symbol |
| `time_generated` | `datetime` | -- | Timestamp when the signal was generated |
| `strategy_id` | `str` | -- | Strategy identifier (maps to MT5 magic number; must be digit string) |
| `forecast` | `Optional[float]` | `0.0` | Signal strength, intended range -20 to +20 |
| `signal_type` | `SignalType` | -- | `BUY` or `SELL` |
| `order_type` | `OrderType` | -- | `MARKET`, `LIMIT`, `STOP`, or `CONT` |
| `order_price` | `Optional[Decimal]` | `Decimal('0.0')` | Limit/stop price (ignored for MARKET orders) |
| `sl` | `Optional[Decimal]` | `Decimal('0.0')` | Stop-loss price |
| `tp` | `Optional[Decimal]` | `Decimal('0.0')` | Take-profit price |
| `rollover` | `Optional[tuple]` | `(False, "", "")` | Rollover info: `(needs_rollover, original_contract, new_contract)` |

---

### `OrderEvent(EventBase)`

```python
class OrderEvent(EventBase):
    type: EventType = EventType.ORDER
    symbol: str
    time_generated: datetime
    strategy_id: str
    volume: Decimal
    signal_type: SignalType
    order_type: OrderType
    order_price: Optional[Decimal] = Decimal('0.0')
    sl: Optional[Decimal] = Decimal('0.0')
    tp: Optional[Decimal] = Decimal('0.0')
    rollover: Optional[tuple] = (False, "", "")
    buffer_data: Optional[dict] = None
```

**Description**: Emitted after the sizing and risk engines approve a signal. The `forecast` from `SignalEvent` has been converted into a concrete `volume`. Consumed by `ExecutionEngine`.

**Attributes**:

| Attribute | Type | Default | Description |
|---|---|---|---|
| `type` | `EventType` | `EventType.ORDER` | Event discriminator |
| `symbol` | `str` | -- | Trading instrument symbol |
| `time_generated` | `datetime` | -- | Timestamp of order creation |
| `strategy_id` | `str` | -- | Strategy identifier |
| `volume` | `Decimal` | -- | Position size (lots) |
| `signal_type` | `SignalType` | -- | `BUY` or `SELL` |
| `order_type` | `OrderType` | -- | `MARKET`, `LIMIT`, `STOP`, or `CONT` |
| `order_price` | `Optional[Decimal]` | `Decimal('0.0')` | Limit/stop price |
| `sl` | `Optional[Decimal]` | `Decimal('0.0')` | Stop-loss price |
| `tp` | `Optional[Decimal]` | `Decimal('0.0')` | Take-profit price |
| `rollover` | `Optional[tuple]` | `(False, "", "")` | Rollover info |
| `buffer_data` | `Optional[dict]` | `None` | Arbitrary extra data passed through to the execution engine |

---

### `FillEvent(EventBase)`

```python
class FillEvent(EventBase):
    type: EventType = EventType.FILL
    deal: DealType
    symbol: str
    time_generated: datetime
    position_id: int
    strategy_id: str
    exchange: str
    volume: Decimal
    price: Decimal
    signal_type: SignalType
    commission: Decimal
    swap: Decimal
    fee: Decimal
    gross_profit: Decimal
    ccy: str
```

**Description**: Emitted by `ExecutionEngine` after an order has been filled (either by the MT5 simulator or live broker). Contains full execution details including costs.

**Attributes**:

| Attribute | Type | Description |
|---|---|---|
| `type` | `EventType` | `EventType.FILL` |
| `deal` | `DealType` | `IN` (entry) or `OUT` (exit) |
| `symbol` | `str` | Trading instrument |
| `time_generated` | `datetime` | Fill timestamp |
| `position_id` | `int` | Broker-assigned position identifier |
| `strategy_id` | `str` | Strategy identifier |
| `exchange` | `str` | Exchange or broker name |
| `volume` | `Decimal` | Filled volume |
| `price` | `Decimal` | Execution price |
| `signal_type` | `SignalType` | Direction of the original signal |
| `commission` | `Decimal` | Commission charged |
| `swap` | `Decimal` | Swap cost |
| `fee` | `Decimal` | Additional fees |
| `gross_profit` | `Decimal` | Gross profit (for exit fills) |
| `ccy` | `str` | Currency denomination for costs and profits |

---

### `ScheduledEvent(EventBase)`

```python
class ScheduledEvent(EventBase):
    type: EventType = Field(default=EventType.SCHEDULED_EVENT, init_var=False)
    schedule_timeframe: StrategyTimeframes
    symbol: str
    timestamp: pd.Timestamp
    former_execution_timestamp: pd.Timestamp | None = None
```

**Description**: Passed to user-registered scheduled callbacks. Created by `ScheduleService` when a timeframe boundary is crossed.

**Attributes**:

| Attribute | Type | Default | Description |
|---|---|---|---|
| `type` | `EventType` | `EventType.SCHEDULED_EVENT` | Event discriminator (not user-settable) |
| `schedule_timeframe` | `StrategyTimeframes` | -- | The timeframe that triggered this event |
| `symbol` | `str` | -- | Symbol from the triggering `BarEvent` |
| `timestamp` | `pd.Timestamp` | -- | Current execution timestamp |
| `former_execution_timestamp` | `pd.Timestamp \| None` | `None` | Timestamp of the previous execution of this schedule |

## Data Flow

```
DataProvider
    |
    v
BarEvent --> TradingDirector._handle_bar_event
                |
                +--> ScheduleService creates ScheduledEvent (passed directly to callbacks, not queued)
                |
                +--> SignalEngineService emits SignalEvent --> queue
                        |
                        v
                    TradingDirector._handle_signal_event
                        |
                        v
                    PortfolioHandler (sizing + risk) emits OrderEvent --> queue
                        |
                        v
                    TradingDirector._handle_order_event
                        |
                        v
                    ExecutionEngine emits FillEvent --> queue
                        |
                        v
                    TradingDirector._handle_fill_event
                        |
                        v
                    PortfolioHandler updates state
```

## Gaps & Issues

1. **No `__all__` definition**: Star-import in `__init__.py` will leak imported names (`BaseModel`, `Decimal`, `pd`, `datetime`, `field`, etc.) into the `pyeventbt.events` namespace.
2. **`rollover` field is an untyped tuple**: A `NamedTuple` or dedicated model would make the `(bool, str, str)` structure self-documenting and type-safe.
3. **`Bar.__price_factor` default is `None` but typed as `float`**: The `field(default=None)` creates an `Optional[float]` at runtime, which is inconsistent with the `float` annotation.
4. **`ScheduledEvent` is not dispatched through the event queue**: Unlike all other events, it is created inline by `ScheduleService` and passed directly to callbacks. Despite having `EventType.SCHEDULED_EVENT`, the `TradingDirector.event_handlers_dict` has no mapping for it.
5. **`forecast` range is documented as -20 to +20 in a comment only**: No validation enforces this range.
6. **Typo in source comment**: "comppact" and "arount" on line 51.

## Requirements Derived

| ID | Requirement | Source |
|---|---|---|
| EVT-01 | All events must inherit from `EventBase` and carry an `EventType` discriminator | `EventBase` class design |
| EVT-02 | Bar prices must be stored as integers with a `digits` field for decimal reconstruction | `Bar` dataclass + properties |
| EVT-03 | `SignalEvent` must carry direction (`SignalType`), order type (`OrderType`), and optional price levels (SL, TP, order price) | `SignalEvent` fields |
| EVT-04 | `OrderEvent` must include a concrete `volume` (converted from forecast by sizing engine) | `OrderEvent.volume` field |
| EVT-05 | `FillEvent` must include full cost breakdown (commission, swap, fee, gross_profit) and currency denomination | `FillEvent` fields |
| EVT-06 | `ScheduledEvent` must reference the triggering timeframe and provide both current and former timestamps | `ScheduledEvent` fields |
| EVT-07 | Rollover support requires a tuple of `(needs_rollover, original_contract, new_contract)` on signal and order events | `rollover` field on `SignalEvent`/`OrderEvent` |
