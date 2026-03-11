# pyeventbt.risk_engine.core.interfaces.risk_engine_interface

## File
`pyeventbt/risk_engine/core/interfaces/risk_engine_interface.py`

## Module
`pyeventbt.risk_engine.core.interfaces.risk_engine_interface`

## Purpose
Defines the `IRiskEngine` protocol that all concrete risk engine implementations must satisfy. This interface establishes the contract for evaluating a `SuggestedOrder` and returning an approved volume.

## Tags
`interface`, `protocol`, `risk-management`, `abstract`

## Dependencies

| Dependency | Usage |
|---|---|
| `typing.Protocol` | Used as base class (though the class does not explicitly inherit `Protocol` -- it uses a plain class with `raise NotImplementedError`) |
| `pyeventbt.strategy.core.modules.Modules` | Passed to `assess_order` for access to runtime state |
| `pyeventbt.portfolio_handler.core.entities.suggested_order.SuggestedOrder` | The order proposal being evaluated |

## Classes/Functions

### `IRiskEngine`

| Field | Value |
|---|---|
| **Signature** | `class IRiskEngine(Protocol)` |
| **Description** | Protocol defining portfolio-level risk management. Implementations evaluate a suggested order and return a float representing the approved volume. A return of 0.0 or less means the order is rejected. |
| **Attributes** | None |
| **Returns** | N/A |

#### `assess_order`

| Field | Value |
|---|---|
| **Signature** | `def assess_order(self, suggested_order: SuggestedOrder, modules: Modules) -> float` |
| **Description** | Evaluates a suggested order against risk constraints. Returns the approved trading volume (may be the original volume, a reduced volume, or 0.0 to reject). |
| **Parameters** | `suggested_order` -- the order proposal from the sizing engine; `modules` -- runtime access to data provider, portfolio, execution engine |
| **Returns** | `float` -- the approved volume; values <= 0 cause the order to be discarded |

## Data Flow

```
SuggestedOrder + Modules
    --> IRiskEngine.assess_order()
    --> float (approved volume)
```

## Gaps & Issues

1. **Declared as `Protocol` but uses `raise NotImplementedError()`.** The `typing.Protocol` import suggests structural subtyping, but the method body raises `NotImplementedError` rather than using `...`. This is inconsistent -- implementations work via nominal subtyping (inheritance) in practice.
2. **Return type could be `Decimal`.** The rest of the codebase uses `Decimal` for volumes, but this interface returns `float`, creating a type inconsistency.

## Requirements Derived

- R-RISK-IF-01: All risk engines shall implement `assess_order(suggested_order, modules) -> float`.
- R-RISK-IF-02: A return value of 0.0 or less shall cause the order to be discarded by the calling service.
- R-RISK-IF-03: The `Modules` parameter shall provide read access to portfolio state, data provider, and execution engine for informed risk decisions.
