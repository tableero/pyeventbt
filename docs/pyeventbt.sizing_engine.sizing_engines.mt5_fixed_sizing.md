# pyeventbt.sizing_engine.sizing_engines.mt5_fixed_sizing

- **File**: `pyeventbt/sizing_engine/sizing_engines/mt5_fixed_sizing.py`
- **Module**: `pyeventbt.sizing_engine.sizing_engines.mt5_fixed_sizing`
- **Purpose**: Implements fixed-volume position sizing. Every order receives the same user-configured lot size, regardless of symbol, account balance, or market conditions.
- **Tags**: `sizing-engine`, `fixed-volume`, `simple`

## Dependencies

| Dependency | Purpose |
|---|---|
| `pyeventbt.sizing_engine.core.interfaces.sizing_engine_interface.ISizingEngine` | Protocol this class satisfies |
| `pyeventbt.sizing_engine.core.configurations.sizing_engine_configurations.FixedSizingConfig` | Configuration providing the fixed volume |
| `pyeventbt.events.events.SignalEvent` | Input event type |
| `pyeventbt.portfolio_handler.core.entities.suggested_order.SuggestedOrder` | Output entity |
| `decimal.Decimal` | Volume precision handling |

## Classes/Functions

### `MT5FixedSizing`

```python
class MT5FixedSizing(ISizingEngine)
```

- **Description**: Sizing engine that applies a constant volume to every order. The volume is read from `FixedSizingConfig` at construction time and stored as a `Decimal`.

#### `__init__`

```python
def __init__(self, configs: FixedSizingConfig) -> None
```

- **Description**: Stores the fixed volume from the configuration.
- **Parameters**:
  - `configs` (`FixedSizingConfig`): Must contain `volume` field.
- **Attributes set**:
  - `self.fixed_volume_size` (`Decimal`): The lot size to use for all orders.

#### `get_suggested_order`

```python
def get_suggested_order(self, signal_event: SignalEvent, *args, **kwargs) -> SuggestedOrder
```

- **Description**: Wraps the signal event into a `SuggestedOrder` using the pre-configured fixed volume.
- **Parameters**:
  - `signal_event` (`SignalEvent`): The trading signal.
  - `*args, **kwargs`: Accepted but ignored.
- **Returns**: `SuggestedOrder` with `volume` set to `self.fixed_volume_size`.

## Data Flow

```
FixedSizingConfig.volume (set once at init)
  |
  v
SuggestedOrder(signal_event, volume=fixed_volume_size)
```

This is the simplest sizing engine -- no runtime queries to the broker or data provider are needed.

## Gaps & Issues

1. **No volume validation**: The engine does not check whether the fixed volume respects the symbol's `volume_min`, `volume_max`, or `volume_step` constraints. Orders with invalid volumes may be rejected by the execution engine or broker.
2. **Signature mismatch with Protocol**: Uses `*args, **kwargs` instead of the explicit `modules: Modules` parameter defined in `ISizingEngine`.

## Requirements Derived

- R-FIXED-SIZING-01: The engine must use exactly the volume specified in `FixedSizingConfig` for every order.
- R-FIXED-SIZING-02: Consider adding validation against symbol volume constraints (`volume_min`, `volume_max`, `volume_step`) to prevent broker rejections.
