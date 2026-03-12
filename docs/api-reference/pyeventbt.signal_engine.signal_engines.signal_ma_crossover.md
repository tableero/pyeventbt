# pyeventbt.signal_engine.signal_engines.signal_ma_crossover

- **File**: `pyeventbt/signal_engine/signal_engines/signal_ma_crossover.py`
- **Module**: `pyeventbt.signal_engine.signal_engines.signal_ma_crossover`
- **Purpose**: Implements a moving average crossover signal engine. Generates BUY signals when the fast MA crosses above the slow MA (and no long position is open), and SELL signals on the inverse crossover. Handles position closure of the opposite side before emitting a new signal.
- **Tags**: `signal-engine`, `moving-average`, `crossover`, `trading-strategy`, `backtest`, `live`

## Dependencies

| Dependency | Type |
|---|---|
| `pyeventbt.strategy.core.modules.Modules` | Internal |
| `pyeventbt.signal_engine.core.interfaces.signal_engine_interface.ISignalEngine` | Internal |
| `pyeventbt.signal_engine.core.configurations.signal_engine_configurations.MACrossoverConfig` | Internal |
| `pyeventbt.data_provider.core.interfaces.data_provider_interface.IDataProvider` | Internal |
| `pyeventbt.portfolio.portfolio.Portfolio` | Internal |
| `pyeventbt.execution_engine.core.interfaces.execution_engine_interface.IExecutionEngine` | Internal |
| `pyeventbt.events.events.BarEvent` | Internal |
| `pyeventbt.events.events.SignalEvent` | Internal |
| `pyeventbt.trading_context.trading_context.TypeContext` | Internal |
| `pandas` | Third-party |
| `datetime.datetime` | Standard library |

## Classes/Functions

### `SignalMACrossover(ISignalEngine)`

- **Signature**: `class SignalMACrossover(ISignalEngine)`
- **Description**: Concrete signal engine that generates trading signals based on a moving average crossover strategy. Supports both simple and exponential moving averages.

#### `__init__`

- **Signature**: `def __init__(self, configurations: MACrossoverConfig, trading_context: trading_context.TypeContext = trading_context.TypeContext.BACKTEST) -> None`
- **Description**: Initialises the engine from a `MACrossoverConfig`, extracting strategy ID, MA type, timeframe, and period lengths.
- **Attributes set**:
  - `self.trading_context` -- `TypeContext` enum value (BACKTEST or LIVE).
  - `self.strategy_id: str` -- Strategy identifier.
  - `self.ma_type: MAType` -- Moving average type (SIMPLE or EXPONENTIAL).
  - `self.signal_timeframe: str` -- Timeframe the engine listens to.
  - `self.fast_period: int | HyperParameter` -- Fast MA period.
  - `self.slow_period: int | HyperParameter` -- Slow MA period.

#### `generate_signal`

- **Signature**: `def generate_signal(self, bar_event: BarEvent, modules: Modules) -> SignalEvent`
- **Description**: Core signal generation logic. Steps:
  1. Filters out bar events that do not match `self.signal_timeframe`.
  2. Fetches the latest `slow_period + 1` bars from the data provider.
  3. Queries the portfolio for current open positions (long/short counts).
  4. Returns early if insufficient data is available (< 2 rows).
  5. Computes fast and slow MAs (simple mean or pandas EWM).
  6. If fast > slow and no long position exists, emits a BUY signal (closing any short positions first).
  7. If fast < slow and no short position exists, emits a SELL signal (closing any long positions first).
  8. Constructs a `SignalEvent` with `forecast=10`, `order_type="MARKET"`, and the latest tick price.
- **Returns**: `SignalEvent` or `None` (implicit return when no signal condition is met or data is insufficient).

## Data Flow

```
BarEvent (from TradingDirector via SignalEngineService)
  |
  v
generate_signal(bar_event, modules)
  |
  +--> modules.DATA_PROVIDER.get_latest_bars(symbol, timeframe, slow_period + 1)
  +--> modules.PORTFOLIO.get_number_of_strategy_open_positions_by_symbol(symbol)
  |
  v
Compute fast_ma, slow_ma (simple or exponential)
  |
  v
Compare MAs + check open positions
  |
  +--> (optional) modules.EXECUTION_ENGINE.close_strategy_{short,long}_positions_by_symbol(symbol)
  |
  v
SignalEvent(symbol, time_generated, strategy_id, forecast=10, signal_type, order_type="MARKET", order_price)
  |
  v
Returned to SignalEngineService --> enqueued
```

## Gaps & Issues

1. `self.trading_context` is compared against the string `"BACKTEST"` (line 100) rather than the enum `TypeContext.BACKTEST`. This works only because `TypeContext` likely has a string representation that matches, but it is fragile and inconsistent.
2. `self.ma_type` is compared against raw strings `"SIMPLE"` / `"EXPONENTIAL"` instead of the `MAType` enum members. Same fragility concern.
3. `forecast` is hardcoded to `10` with a comment stating "Average forecast as this is a discrete signal strategy" -- this value has no configurability.
4. The engine directly calls `modules.EXECUTION_ENGINE.close_strategy_{short,long}_positions_by_symbol()` as a side effect during signal generation. This couples signal generation to execution, violating the separation of concerns in the event-driven architecture.
5. The `local variable 'signal' might be referenced before assignment` scenario: if `self.ma_type` is neither `"SIMPLE"` nor `"EXPONENTIAL"`, the MA variables (`fast_ma`, `slow_ma`) are undefined, leading to a `NameError`.
6. Uses `pandas` for MA calculations despite the framework's stated preference for `polars` DataFrames and Numba-accelerated indicators.
7. No handling of `HyperParameter` unwrapping -- if `fast_period` or `slow_period` is a `HyperParameter` object, arithmetic and slicing operations will fail.

## Requirements Derived

- REQ-SIGMAC-01: The engine must only process bar events matching the configured `signal_timeframe`.
- REQ-SIGMAC-02: BUY signals must only be emitted when no long position is currently open for the symbol.
- REQ-SIGMAC-03: SELL signals must only be emitted when no short position is currently open for the symbol.
- REQ-SIGMAC-04: Opposite-side positions must be closed before emitting a new directional signal.
- REQ-SIGMAC-05: The engine must support both SIMPLE and EXPONENTIAL moving average types.
- REQ-SIGMAC-06: Signal generation must degrade gracefully when insufficient historical data is available.
