# pyeventbt.sizing_engine.sizing_engines.mt5_min_sizing

- **File**: `pyeventbt/sizing_engine/sizing_engines/mt5_min_sizing.py`
- **Module**: `pyeventbt.sizing_engine.sizing_engines.mt5_min_sizing`
- **Purpose**: Implements the minimum-lot sizing strategy. Queries the MT5 broker (simulated or live) for the symbol's `volume_min` and uses that as the order volume.
- **Tags**: `sizing-engine`, `mt5`, `minimum-lot`, `backtest`, `live`

## Dependencies

| Dependency | Purpose |
|---|---|
| `pyeventbt.utils.utils.check_platform_compatibility` | Validates platform before importing live MT5 |
| `pyeventbt.sizing_engine.core.interfaces.sizing_engine_interface.ISizingEngine` | Protocol this class satisfies |
| `pyeventbt.events.events.SignalEvent` | Input event type |
| `pyeventbt.portfolio_handler.core.entities.suggested_order.SuggestedOrder` | Output entity |
| `pyeventbt.trading_context.trading_context.TypeContext` | Enum for `BACKTEST` / `LIVE` context |
| `pyeventbt.broker.mt5_broker.mt5_simulator_wrapper.Mt5SimulatorWrapper` | Backtest-mode MT5 shim (conditionally imported) |
| `MetaTrader5` | Live MT5 Python package (conditionally imported) |
| `decimal.Decimal` | Volume precision handling |

## Classes/Functions

### `MT5MinSizing`

```python
class MT5MinSizing(ISizingEngine)
```

- **Description**: Sizing engine that always uses the broker-defined minimum volume for the traded symbol. Suitable for testing or conservative position sizing.

#### `__init__`

```python
def __init__(self, trading_context: TypeContext = TypeContext.BACKTEST) -> None
```

- **Description**: Conditionally imports the MT5 connector based on context. In backtest mode, uses `Mt5SimulatorWrapper`; in live mode, imports `MetaTrader5`.
- **Parameters**:
  - `trading_context` (`TypeContext`): Defaults to `TypeContext.BACKTEST`.
- **Attributes set**:
  - `self.mt5`: The MT5 module/class used for symbol queries.

#### `get_suggested_order`

```python
def get_suggested_order(self, signal_event: SignalEvent, *args, **kwargs) -> SuggestedOrder
```

- **Description**: Queries `self.mt5.symbol_info(symbol).volume_min` and wraps the signal event into a `SuggestedOrder` with that volume.
- **Parameters**:
  - `signal_event` (`SignalEvent`): Signal containing the symbol name and trade direction.
  - `*args, **kwargs`: Accepted but ignored (allows `modules` to be passed without error).
- **Returns**: `SuggestedOrder` with `volume` set to the symbol's minimum lot size as `Decimal`.

## Data Flow

```
SignalEvent.symbol
  |
  v
mt5.symbol_info(symbol).volume_min
  |
  v
SuggestedOrder(signal_event, volume=Decimal(volume_min))
```

## Gaps & Issues

1. **String comparison for context**: The `__init__` compares `trading_context == "BACKTEST"` (a string literal) rather than using the `TypeContext.BACKTEST` enum member. This works because the enum's value is the string `"BACKTEST"`, but it is fragile.
2. **`mt5` set to `None` on import failure**: If `MetaTrader5` is not installed in live mode, `self.mt5` is set to `None`, which will cause an `AttributeError` on the first call to `get_suggested_order`.
3. **Signature mismatch with Protocol**: The Protocol defines `get_suggested_order(self, signal_event, modules)` but this class uses `*args, **kwargs` instead of the explicit `modules` parameter.

## Requirements Derived

- R-MIN-SIZING-01: The engine must return the broker's minimum allowed volume for the given symbol.
- R-MIN-SIZING-02: The engine must function in both backtest and live contexts by loading the appropriate MT5 connector.
