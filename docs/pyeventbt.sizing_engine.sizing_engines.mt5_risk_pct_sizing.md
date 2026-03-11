# pyeventbt.sizing_engine.sizing_engines.mt5_risk_pct_sizing

- **File**: `pyeventbt/sizing_engine/sizing_engines/mt5_risk_pct_sizing.py`
- **Module**: `pyeventbt.sizing_engine.sizing_engines.mt5_risk_pct_sizing`
- **Purpose**: Implements risk-percentage-based position sizing. Calculates order volume so that if the stop-loss is hit, the monetary loss equals a specified percentage of account equity. Handles cross-currency conversion for non-account-denominated instruments.
- **Tags**: `sizing-engine`, `risk-management`, `mt5`, `currency-conversion`, `backtest`, `live`

## Dependencies

| Dependency | Purpose |
|---|---|
| `pyeventbt.strategy.core.modules.Modules` | Provides `DATA_PROVIDER` for tick data and currency conversion |
| `pyeventbt.sizing_engine.core.interfaces.sizing_engine_interface.ISizingEngine` | Protocol this class satisfies |
| `pyeventbt.sizing_engine.core.configurations.sizing_engine_configurations.RiskPctSizingConfig` | Configuration providing `risk_pct` |
| `pyeventbt.events.events.SignalEvent` | Input event type |
| `pyeventbt.data_provider.core.interfaces.data_provider_interface.IDataProvider` | Imported for type reference |
| `pyeventbt.portfolio_handler.core.entities.suggested_order.SuggestedOrder` | Output entity |
| `pyeventbt.trading_context.trading_context.TypeContext` | Enum for `BACKTEST` / `LIVE` context |
| `pyeventbt.utils.utils.Utils` | `convert_currency_amount_to_another_currency` (static utility used in `get_suggested_order`) |
| `pyeventbt.utils.utils.check_platform_compatibility` | Platform check before live MT5 import |
| `pyeventbt.broker.mt5_broker.mt5_simulator_wrapper.Mt5SimulatorWrapper` | Backtest-mode MT5 shim (conditionally imported) |
| `MetaTrader5` | Live MT5 Python package (conditionally imported) |
| `decimal.Decimal` | Precision handling for risk percentage and volume |

## Classes/Functions

### `MT5RiskPctSizing`

```python
class MT5RiskPctSizing(ISizingEngine)
```

- **Description**: The most sophisticated sizing engine in the package. Calculates position size from account equity, risk percentage, stop-loss distance, symbol tick properties, and cross-currency conversion rates.

#### `__init__`

```python
def __init__(self, configs: RiskPctSizingConfig, trading_context: TypeContext = TypeContext.BACKTEST) -> None
```

- **Description**: Stores risk percentage and conditionally imports the MT5 connector.
- **Parameters**:
  - `configs` (`RiskPctSizingConfig`): Must contain `risk_pct`.
  - `trading_context` (`TypeContext`): Defaults to `TypeContext.BACKTEST`.
- **Attributes set**:
  - `self.mt5`: MT5 module/class for account and symbol info queries.
  - `self.risk_pct` (`float`): Risk percentage per trade.

#### `_convert_currency_amount_to_another_currency`

```python
def _convert_currency_amount_to_another_currency(self, amount: float, from_ccy: str, to_ccy: str, latest_tick: dict) -> float
```

- **Description**: Converts a monetary amount between two currencies using a hard-coded list of 30 major FX pairs and the latest tick bid price. Determines whether to multiply or divide based on which currency is the base of the FX pair.
- **Parameters**:
  - `amount` (`float`): Amount to convert.
  - `from_ccy` (`str`): Source currency code (e.g., `"USD"`).
  - `to_ccy` (`str`): Target currency code (e.g., `"EUR"`).
  - `latest_tick` (`dict`): Dictionary with at least a `'bid'` key.
- **Returns**: `float` -- converted amount.
- **Note**: This method appears to be **dead code**. The actual `get_suggested_order` method calls `Utils.convert_currency_amount_to_another_currency` instead.

#### `get_suggested_order`

```python
def get_suggested_order(self, signal_event: SignalEvent, modules: Modules) -> SuggestedOrder
```

- **Description**: Core sizing logic. Calculates volume using the formula:
  1. Determine entry price (ask for BUY market orders, bid for SELL, or explicit order price for limit/stop orders).
  2. Compute `tick_value_profit_ccy = contract_size * tick_size`.
  3. Convert tick value from profit currency to account currency via `Utils.convert_currency_amount_to_another_currency`.
  4. `price_distance = abs(entry_price - sl) / tick_size` (integer tick count).
  5. `monetary_risk = equity * risk_pct / 100`.
  6. `volume = monetary_risk / (price_distance * tick_value_account_ccy)`.
  7. Round volume to the nearest `volume_step`.
- **Parameters**:
  - `signal_event` (`SignalEvent`): Must have `symbol`, `signal_type` (`"BUY"` / `"SELL"`), `order_type` (`"MARKET"` or other), `sl` (stop-loss price, non-zero), and optionally `order_price`.
  - `modules` (`Modules`): Must provide `DATA_PROVIDER` with `get_latest_tick(symbol)`.
- **Returns**: `SuggestedOrder` with calculated `volume`.
- **Raises**:
  - `Exception` if `risk_pct <= 0`.
  - `Exception` if `signal_event.sl == 0` (stop-loss required).

## Data Flow

```
SignalEvent + Modules.DATA_PROVIDER
  |
  v
mt5.account_info()  -->  equity, account_currency
mt5.symbol_info()   -->  volume_step, tick_size, contract_size, currency_profit
DATA_PROVIDER.get_latest_tick()  -->  bid/ask for entry price estimation
  |
  v
tick_value_profit_ccy = contract_size * tick_size
tick_value_account_ccy = Utils.convert(tick_value, profit_ccy, account_ccy, DATA_PROVIDER)
price_distance = |entry - sl| / tick_size
volume = (equity * risk_pct / 100) / (price_distance * tick_value_account_ccy)
volume = round(volume / volume_step) * volume_step
  |
  v
SuggestedOrder(signal_event, volume)
```

## Gaps & Issues

1. **Hard-coded FX pair list**: `_convert_currency_amount_to_another_currency` contains only 30 pairs. Any symbol with a profit currency not covered (e.g., TRY, ZAR, HKD, SGD, MXN) will cause an `IndexError` from the empty list comprehension.
2. **Dead code**: The instance method `_convert_currency_amount_to_another_currency` is defined but never called; `get_suggested_order` uses `Utils.convert_currency_amount_to_another_currency` instead.
3. **String comparison for context**: `trading_context == "BACKTEST"` compares against a string literal rather than the enum member.
4. **`mt5` may be `None`**: If `MetaTrader5` fails to import in live mode, `self.mt5 = None` and subsequent calls to `account_info()` / `symbol_info()` will raise `AttributeError`.
5. **Integer cast on price distance**: `int(abs(entry_price - signal_event.sl) / tick_size)` truncates fractional ticks, which can underestimate the stop-loss distance and oversize the position.
6. **Mixed Decimal/float arithmetic**: `risk_pct` is stored as `float`, converted to `Decimal` locally, but then mixed with float values from `mt5.account_info()` and `mt5.symbol_info()`. This may cause type errors or precision loss depending on the MT5 wrapper implementation.
7. **No volume bounds check**: The calculated volume is not clamped to `volume_min` / `volume_max`.

## Requirements Derived

- R-RISKPCT-01: The engine must require a non-zero stop-loss on every signal event.
- R-RISKPCT-02: The engine must convert tick value to account currency when the symbol's profit currency differs from the account currency.
- R-RISKPCT-03: The calculated volume must be rounded to the symbol's `volume_step`.
- R-RISKPCT-04: The engine should validate that the final volume falls within `[volume_min, volume_max]` before returning.
- R-RISKPCT-05: Currency conversion should support a broader set of FX pairs or use a dynamic lookup mechanism.
