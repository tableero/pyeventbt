# pyeventbt.risk_engine

## Package
`pyeventbt.risk_engine`

## Purpose
Top-level package for the risk engine module. Provides portfolio-level risk management that evaluates suggested orders before they become actual order events. Re-exports key configuration and engine classes for convenient access.

## Tags
`risk-management`, `order-filtering`, `portfolio-level`, `event-pipeline`

## Modules

| Submodule | Description |
|---|---|
| `core.configurations.risk_engine_configurations` | Pydantic configuration models for risk engine variants |
| `core.interfaces.risk_engine_interface` | `IRiskEngine` protocol defining the risk engine contract |
| `services.risk_engine_service` | `RiskEngineService` orchestrator that delegates to concrete engines and enqueues `OrderEvent`s |
| `risk_engines.passthrough_risk_engine` | `PassthroughRiskEngine` -- a no-op implementation that approves all orders unchanged |

## Internal Architecture

The risk engine sits between the sizing engine and the execution engine in the event pipeline. When `PortfolioHandler` receives a `SignalEvent`, it first runs the sizing engine to produce a `SuggestedOrder`, then passes that to `RiskEngineService.assess_order()`. The service delegates to the configured concrete engine (currently only `PassthroughRiskEngine`), receives back a volume, and if the volume is positive, creates an `OrderEvent` and puts it on the shared events queue.

Custom risk logic can be injected at runtime via `RiskEngineService.set_custom_asses_order()`, which replaces the `assess_order` method with a user-supplied callable.

```
SuggestedOrder --> RiskEngineService.assess_order()
                      |
                      +--> concrete IRiskEngine.assess_order() --> volume (float)
                      |
                      +--> if volume > 0: create OrderEvent --> events_queue.put()
```

## Cross-Package Dependencies

| Dependency | Usage |
|---|---|
| `pyeventbt.strategy.core.modules.Modules` | Passed to concrete risk engines for access to portfolio/data state |
| `pyeventbt.portfolio_handler.core.entities.suggested_order.SuggestedOrder` | Input entity evaluated by the risk engine |
| `pyeventbt.events.events.OrderEvent` | Output event created when an order passes risk checks |
| `pydantic.BaseModel` | Base for configuration classes |
| `queue.Queue` | Shared event queue for emitting `OrderEvent`s |

## Gaps & Issues

1. **Only one concrete engine exists.** `PassthroughRiskEngine` performs no actual risk filtering; it always returns the suggested volume unchanged. No max-drawdown, exposure-limit, or correlation-based risk engine is provided.
2. **Fallback in factory is redundant.** `RiskEngineService._get_risk_management_method` returns `PassthroughRiskEngine()` in both the `isinstance(PassthroughRiskConfig)` branch and the `else` branch, so any unrecognized config silently becomes a passthrough rather than raising an error.
3. **Typo in public method name.** `set_custom_asses_order` should be `set_custom_assess_order`.
4. **`__init__.py` re-exports.** Only `PassthroughRiskConfig` and `PassthroughRiskEngine` are exported; `BaseRiskConfig` and `RiskEngineService` are not, which may require users to reach into submodules.
