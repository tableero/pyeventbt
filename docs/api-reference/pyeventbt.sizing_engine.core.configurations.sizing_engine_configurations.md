# pyeventbt.sizing_engine.core.configurations.sizing_engine_configurations

- **File**: `pyeventbt/sizing_engine/core/configurations/sizing_engine_configurations.py`
- **Module**: `pyeventbt.sizing_engine.core.configurations.sizing_engine_configurations`
- **Purpose**: Defines Pydantic configuration models for each position sizing strategy. These configs are passed to `SizingEngineService` to select and parameterise the appropriate sizing engine.
- **Tags**: `configuration`, `pydantic`, `sizing`, `position-management`

## Dependencies

| Dependency | Purpose |
|---|---|
| `pydantic.BaseModel` | Base class for all config models |
| `decimal.Decimal` | Used for the `volume` field in `FixedSizingConfig` |

## Classes/Functions

### `BaseSizingConfig`

```python
class BaseSizingConfig(BaseModel)
```

- **Description**: Abstract base configuration for all sizing strategies. Contains no fields; acts as a type discriminator for `SizingEngineService._get_position_sizing_method`. When used directly, the service defaults to `MT5MinSizing`.
- **Attributes**: None
- **Returns**: N/A

---

### `MinSizingConfig`

```python
class MinSizingConfig(BaseSizingConfig)
```

- **Description**: Configuration that selects the minimum-lot sizing strategy. The engine will query the broker for the symbol's `volume_min` at order time.
- **Attributes**: None (inherits from `BaseSizingConfig`)
- **Returns**: N/A

---

### `FixedSizingConfig`

```python
class FixedSizingConfig(BaseSizingConfig)
```

- **Description**: Configuration for fixed-volume sizing. Every order placed with this config uses the same lot size regardless of account balance or market conditions.
- **Attributes**:
  - `volume` (`Decimal`): Fixed volume/quantity for all positions.
- **Returns**: N/A

---

### `RiskPctSizingConfig`

```python
class RiskPctSizingConfig(BaseSizingConfig)
```

- **Description**: Configuration for risk-percentage-based sizing. The engine calculates position size so that if the stop-loss is hit, the loss equals `risk_pct`% of account equity.
- **Attributes**:
  - `risk_pct` (`float`): Risk percentage of account equity per trade. A value of `1` means 1%.
- **Returns**: N/A

## Data Flow

1. User instantiates one of these config classes (e.g., `FixedSizingConfig(volume=0.1)`).
2. Config is passed to `Strategy.backtest()` or `Strategy.run_live()`.
3. `SizingEngineService.__init__` receives the config and calls `_get_position_sizing_method` to select the concrete engine.

## Gaps & Issues

1. **No `sl_pips` field on `RiskPctSizingConfig`**: The code knowledge mentions `sl_pips(float|None)` but the actual source only contains `risk_pct`. Stop-loss information comes from the `SignalEvent` at runtime instead.
2. **No validation on `risk_pct`**: Pydantic validation (e.g., `gt=0`) is not applied; the check is deferred to `MT5RiskPctSizing.get_suggested_order` at runtime.
3. **`BaseSizingConfig` doubles as default**: Passing a bare `BaseSizingConfig()` silently falls through to the `else` branch in the service, defaulting to `MT5MinSizing`. This implicit behaviour could be surprising.

## Requirements Derived

- R-SIZING-CFG-01: All sizing configurations must extend `BaseSizingConfig`.
- R-SIZING-CFG-02: `FixedSizingConfig.volume` must be a `Decimal` to avoid floating-point precision issues.
- R-SIZING-CFG-03: `RiskPctSizingConfig.risk_pct` must be greater than 0; currently enforced at runtime, not at config validation time.
