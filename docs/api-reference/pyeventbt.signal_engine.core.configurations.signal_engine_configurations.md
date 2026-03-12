# pyeventbt.signal_engine.core.configurations.signal_engine_configurations

- **File**: `pyeventbt/signal_engine/core/configurations/signal_engine_configurations.py`
- **Module**: `pyeventbt.signal_engine.core.configurations.signal_engine_configurations`
- **Purpose**: Defines Pydantic configuration models for signal engines, including the base config, MA type enumeration, and the MA crossover-specific config.
- **Tags**: `configuration`, `pydantic`, `signal-engine`, `moving-average`

## Dependencies

| Dependency | Type |
|---|---|
| `pydantic.BaseModel` | Third-party |
| `enum.Enum` | Standard library |
| `pyeventbt.core.entities.hyper_parameter.HyperParameter` | Internal |

## Classes/Functions

### `BaseSignalEngineConfig(BaseModel)`

- **Signature**: `class BaseSignalEngineConfig(BaseModel)`
- **Description**: Base Pydantic model for all signal engine configurations. Provides the minimum fields every signal engine config must carry.
- **Attributes**:
  - `strategy_id: str` -- Unique strategy identifier (maps to MT5 Magic Number; must be a string of digits).
  - `signal_timeframe: str` -- Timeframe string on which the signal engine operates (e.g., `"1h"`, `"4h"`).
- **Returns**: N/A (data model)

### `MAType(str, Enum)`

- **Signature**: `class MAType(str, Enum)`
- **Description**: Enumeration of supported moving average calculation methods.
- **Attributes**:
  - `SIMPLE = "SIMPLE"` -- Simple moving average.
  - `EXPONENTIAL = "EXPONENTIAL"` -- Exponential moving average.
- **Returns**: N/A (enum)

### `MACrossoverConfig(BaseSignalEngineConfig)`

- **Signature**: `class MACrossoverConfig(BaseSignalEngineConfig)`
- **Description**: Configuration model for the MA crossover signal engine. Extends `BaseSignalEngineConfig` with period lengths and MA type.
- **Attributes**:
  - `strategy_id: str` -- (redeclared from base) Strategy identifier.
  - `signal_timeframe: str` -- (redeclared from base) Timeframe for signal generation.
  - `ma_type: MAType` -- Type of moving average; defaults to `MAType.SIMPLE`.
  - `fast_period: int | HyperParameter` -- Period for the fast moving average. Accepts a fixed int or a `HyperParameter` for optimisation sweeps.
  - `slow_period: int | HyperParameter` -- Period for the slow moving average. Accepts a fixed int or a `HyperParameter` for optimisation sweeps.
- **Returns**: N/A (data model)

## Data Flow

```
User code / Strategy
  |
  v
MACrossoverConfig(strategy_id=..., signal_timeframe=..., fast_period=..., slow_period=...)
  |
  v
SignalEngineService.__init__  -->  _get_signal_engine  -->  SignalMACrossover(configurations=...)
```

Configuration objects are passed at strategy construction time and are consumed by `SignalEngineService` to select and initialise the appropriate signal engine.

## Gaps & Issues

1. `MACrossoverConfig` redeclares `strategy_id` and `signal_timeframe` that already exist on `BaseSignalEngineConfig`. This is redundant and could lead to confusion if the base fields ever gain validators.
2. `fast_period` and `slow_period` accept `int | HyperParameter`, but downstream code (`SignalMACrossover`) uses these values directly in arithmetic and slicing without unwrapping the `HyperParameter` -- this will fail at runtime when a `HyperParameter` object is passed unless the optimiser resolves it first.
3. No validation ensures `fast_period < slow_period`.

## Requirements Derived

- REQ-SIGCFG-01: Every signal engine config must provide `strategy_id` and `signal_timeframe`.
- REQ-SIGCFG-02: `MACrossoverConfig` must support both fixed integer periods and `HyperParameter` objects for optimisation.
- REQ-SIGCFG-03: `MAType` must enumerate at least SIMPLE and EXPONENTIAL variants.
