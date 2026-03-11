# pyeventbt.risk_engine.risk_engines.passthrough_risk_engine

## File
`pyeventbt/risk_engine/risk_engines/passthrough_risk_engine.py`

## Module
`pyeventbt.risk_engine.risk_engines.passthrough_risk_engine`

## Purpose
Implements a no-op risk engine that approves all suggested orders without modification. Returns the proposed volume unchanged, effectively bypassing risk filtering.

## Tags
`risk-engine`, `passthrough`, `no-op`, `default`

## Dependencies

| Dependency | Usage |
|---|---|
| `pyeventbt.strategy.core.modules.Modules` | Received but unused parameter |
| `pyeventbt.risk_engine.core.interfaces.risk_engine_interface.IRiskEngine` | Interface being implemented |
| `pyeventbt.portfolio_handler.core.entities.suggested_order.SuggestedOrder` | Input entity whose volume is returned as-is |

## Classes/Functions

### `PassthroughRiskEngine`

| Field | Value |
|---|---|
| **Signature** | `class PassthroughRiskEngine(IRiskEngine)` |
| **Description** | A risk engine that performs no filtering. Every suggested order is approved with its original volume. This is the default and currently only concrete risk engine in the framework. |
| **Attributes** | None |

#### `assess_order`

| Field | Value |
|---|---|
| **Signature** | `def assess_order(self, suggested_order: SuggestedOrder, modules: Modules) -> float` |
| **Description** | Returns `suggested_order.volume` unchanged, approving the order unconditionally. |
| **Parameters** | `suggested_order` -- the order proposal; `modules` -- runtime context (unused) |
| **Returns** | `float` -- the original proposed volume from `suggested_order.volume` |

## Data Flow

```
SuggestedOrder.volume --> PassthroughRiskEngine.assess_order() --> same volume (unchanged)
```

## Gaps & Issues

1. **No risk logic.** This engine is purely a placeholder. It does not check margin availability, exposure limits, drawdown thresholds, or any other risk metric.
2. **`modules` parameter is unused.** Accepted for interface compatibility but never referenced.
3. **Return type ambiguity.** `suggested_order.volume` is likely a `Decimal` (based on other parts of the codebase), but the return type annotation is `float`.

## Requirements Derived

- R-RISK-PT-01: `PassthroughRiskEngine.assess_order` shall return the suggested volume without modification.
- R-RISK-PT-02: The engine shall accept `Modules` for interface compliance but is not required to use it.
