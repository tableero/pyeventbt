# pyeventbt.strategy.core.account_currencies

**File**: `pyeventbt/strategy/core/account_currencies.py`

**Module**: `pyeventbt.strategy.core.account_currencies`

**Purpose**: Defines the `AccountCurrencies` enum representing supported account denomination currencies for backtesting and live trading.

**Tags**: `#enum` `#configuration` `#currency` `#account`

---

## Dependencies

- `enum.Enum` (stdlib)

---

## Classes

### `AccountCurrencies(str, Enum)`

String enum of supported base currencies for trading accounts.

| Member | Value | Description |
|---|---|---|
| `EUR` | `'EUR'` | Euro |
| `USD` | `'USD'` | United States Dollar |
| `GBP` | `'GBP'` | British Pound Sterling |

#### Usage

```python
strategy.backtest(
    account_currency=AccountCurrencies.USD,
    ...
)
```

Used as the `account_currency` parameter in `Strategy.backtest()` and forwarded to `CSVBacktestDataConfig` and `MT5SimulatedExecutionConfig`.

---

## Data Flow

- **Input**: User selects an `AccountCurrencies` member when calling `Strategy.backtest()`.
- **Output**: Propagated to data provider and execution engine configurations for currency conversion calculations during P&L computation.

---

## Gaps & Issues

1. **Very limited currency support**: Only EUR, USD, and GBP are supported. Major trading currencies like JPY, CHF, AUD, CAD, NZD are absent. This limits the framework's applicability for accounts denominated in other currencies.
2. **No mechanism for extension**: Users cannot add custom currencies without modifying this enum directly.

---

## Requirements Derived

1. **REQ-CUR-001**: The system shall support at least EUR, USD, and GBP as account denomination currencies.
2. **REQ-CUR-002**: The account currency shall be used for P&L calculations and position sizing throughout the backtest and live trading pipelines.
