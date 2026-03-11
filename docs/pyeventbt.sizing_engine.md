# pyeventbt.sizing_engine

- **Package**: `pyeventbt.sizing_engine`
- **Purpose**: Provides position sizing strategies that convert `SignalEvent`s into `SuggestedOrder`s with calculated trade volumes. Supports minimum lot, fixed volume, and risk-percentage-based sizing for MT5-compatible instruments.
- **Tags**: `sizing`, `position-management`, `risk`, `mt5`, `order-generation`

## Modules

| Module | Path | Description |
|---|---|---|
| `__init__` | `sizing_engine/__init__.py` | Re-exports configuration classes and all concrete sizing engine implementations |
| `sizing_engine_configurations` | `core/configurations/sizing_engine_configurations.py` | Pydantic config models: `BaseSizingConfig`, `MinSizingConfig`, `FixedSizingConfig`, `RiskPctSizingConfig` |
| `sizing_engine_interface` | `core/interfaces/sizing_engine_interface.py` | `ISizingEngine` Protocol defining the `get_suggested_order` contract |
| `sizing_engine_service` | `services/sizing_engine_service.py` | `SizingEngineService` -- factory and delegation layer that selects and invokes the correct sizing engine |
| `mt5_min_sizing` | `sizing_engines/mt5_min_sizing.py` | `MT5MinSizing` -- uses the broker-defined minimum volume for the symbol |
| `mt5_fixed_sizing` | `sizing_engines/mt5_fixed_sizing.py` | `MT5FixedSizing` -- applies a user-specified fixed volume to every order |
| `mt5_risk_pct_sizing` | `sizing_engines/mt5_risk_pct_sizing.py` | `MT5RiskPctSizing` -- calculates volume from account equity, risk percentage, and stop-loss distance |

## Internal Architecture

```
SizingEngineService
  |
  |-- _get_position_sizing_method(config)
  |       |-- MinSizingConfig    -> MT5MinSizing
  |       |-- FixedSizingConfig  -> MT5FixedSizing
  |       |-- RiskPctSizingConfig-> MT5RiskPctSizing
  |       |-- (default)          -> MT5MinSizing
  |
  |-- get_suggested_order(signal_event)
  |       delegates to selected ISizingEngine
  |
  |-- set_suggested_order_function(fn)
          monkey-patches the engine's method for custom sizing
```

All concrete engines implement `ISizingEngine` (Protocol) and return a `SuggestedOrder` entity. The service is instantiated by `PortfolioHandler` and called during the `SIGNAL -> ORDER` pipeline stage.

## Cross-Package Dependencies

| Dependency | Purpose |
|---|---|
| `pyeventbt.events.events.SignalEvent` | Input event triggering sizing |
| `pyeventbt.portfolio_handler.core.entities.suggested_order.SuggestedOrder` | Output entity returned by all engines |
| `pyeventbt.strategy.core.modules.Modules` | Provides `DATA_PROVIDER`, `EXECUTION_ENGINE`, `PORTFOLIO`, `TRADING_CONTEXT` |
| `pyeventbt.data_provider` | Used by `MT5RiskPctSizing` for latest tick / currency conversion |
| `pyeventbt.broker.mt5_broker.mt5_simulator_wrapper` | Backtest-mode MT5 shim used by `MT5MinSizing` and `MT5RiskPctSizing` |
| `pyeventbt.trading_context.trading_context` | `TypeContext` enum (`BACKTEST` / `LIVE`) |
| `pyeventbt.utils.utils` | `Utils.convert_currency_amount_to_another_currency`, `check_platform_compatibility` |
| `pydantic` | Base for configuration models |

## Gaps & Issues

1. **Exotic pair currency conversion**: `MT5RiskPctSizing._convert_currency_amount_to_another_currency` uses a hard-coded list of 30 FX symbols. Pairs not in that list will raise an `IndexError` when the list comprehension returns empty.
2. **Unused private method**: `_convert_currency_amount_to_another_currency` is defined on the class but actual `get_suggested_order` calls `Utils.convert_currency_amount_to_another_currency` instead -- the private method appears dead code.
3. **`set_suggested_order_function` monkey-patches**: Replacing an engine method at runtime bypasses the Protocol contract and may confuse type checkers.
4. **Duplicate import**: `SuggestedOrder` is imported twice in `sizing_engine_service.py`.
5. **String comparison for context**: `MT5MinSizing` and `MT5RiskPctSizing` compare `trading_context` against the string `"BACKTEST"` rather than the enum `TypeContext.BACKTEST`.
6. **No unit tests**: The repository has no automated tests for any sizing engine.
