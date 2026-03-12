# pyeventbt.risk_engine.core.configurations.risk_engine_configurations

## File
`pyeventbt/risk_engine/core/configurations/risk_engine_configurations.py`

## Module
`pyeventbt.risk_engine.core.configurations.risk_engine_configurations`

## Purpose
Defines Pydantic-based configuration models used to select and parameterize risk engine implementations. The configuration object is passed to `RiskEngineService` at construction time to determine which concrete risk engine to instantiate.

## Tags
`configuration`, `pydantic`, `risk-management`

## Dependencies

| Dependency | Usage |
|---|---|
| `pydantic.BaseModel` | Base class for all config models |

## Classes/Functions

### `BaseRiskConfig`

| Field | Value |
|---|---|
| **Signature** | `class BaseRiskConfig(BaseModel)` |
| **Description** | Abstract base configuration for all risk engines. Contains no fields; serves as a type discriminator in `RiskEngineService._get_risk_management_method`. |
| **Attributes** | None |
| **Returns** | N/A |

### `PassthroughRiskConfig`

| Field | Value |
|---|---|
| **Signature** | `class PassthroughRiskConfig(BaseRiskConfig)` |
| **Description** | Configuration that selects the `PassthroughRiskEngine`, which approves all orders with unchanged volume. No parameters are needed. |
| **Attributes** | None (inherits empty `BaseRiskConfig`) |
| **Returns** | N/A |

## Data Flow

```
User code creates PassthroughRiskConfig()
    --> passed to Strategy
    --> Strategy passes to RiskEngineService.__init__(risk_config=...)
    --> RiskEngineService._get_risk_management_method(risk_config)
        --> isinstance check selects PassthroughRiskEngine
```

## Gaps & Issues

1. **No parameterized risk configs.** There are no configs for common risk strategies (e.g., max position count, max drawdown percentage, exposure limits). Users must use `set_custom_asses_order` for any real risk logic.
2. **No validation.** Since both classes are empty `pass` bodies, there is nothing to validate. Future configs should add fields with Pydantic validators.

## Requirements Derived

- R-RISK-CFG-01: The system shall provide a `BaseRiskConfig` base class from which all risk engine configurations derive.
- R-RISK-CFG-02: `PassthroughRiskConfig` shall require no parameters and map to the passthrough (no-op) risk engine.
- R-RISK-CFG-03: Future risk configurations should support numeric thresholds (e.g., max drawdown, max exposure) validated by Pydantic.
