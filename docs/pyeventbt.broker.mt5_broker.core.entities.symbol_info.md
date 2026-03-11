# File: `pyeventbt/broker/mt5_broker/core/entities/symbol_info.py`

## Module
`pyeventbt.broker.mt5_broker.core.entities.symbol_info`

## Purpose
Defines the `SymbolInfo` Pydantic model representing the full specification of a trading symbol in MetaTrader 5. This is the largest entity model in the broker layer, with approximately 97 fields covering pricing, volume, trade rules, session statistics, Greeks, and metadata. Mirrors the structure returned by `mt5.symbol_info()`.

## Tags
`entity`, `pydantic`, `symbol`, `mt5`, `market-data`, `instrument`

## Dependencies

| Dependency | Type | Purpose |
|---|---|---|
| `pydantic.BaseModel` | External | Base class for validated data model |
| `decimal.Decimal` | Stdlib | Precision type for all price/volume/financial fields |

## Classes/Functions

### `class SymbolInfo(BaseModel)`

Pydantic model representing complete MT5 symbol information.

**Attributes (grouped by category)**:

**Boolean flags**:
| Attribute | Type | Description |
|---|---|---|
| `custom` | `bool` | Whether the symbol is custom-created |
| `select` | `bool` | Whether symbol is selected in MarketWatch |
| `visible` | `bool` | Whether symbol is visible in MarketWatch |
| `spread_float` | `bool` | Whether spread is floating |
| `margin_hedged_use_leg` | `bool` | Whether hedged margin uses larger leg |

**Integer fields**:
| Attribute | Type | Description |
|---|---|---|
| `chart_mode` | `int` | Chart price type (0=BID, 1=LAST) |
| `session_deals` | `int` | Number of deals in current session |
| `session_buy_orders` | `int` | Number of buy orders in current session |
| `session_sell_orders` | `int` | Number of sell orders in current session |
| `time` | `int` | Last quote time (Unix timestamp) |
| `digits` | `int` | Number of decimal places for price |
| `spread` | `int` | Current spread in points |
| `ticks_bookdepth` | `int` | Depth of Market book depth |
| `trade_calc_mode` | `int` | Margin calculation mode |
| `trade_mode` | `int` | Trade execution mode |
| `start_time` | `int` | Symbol trading start date |
| `expiration_time` | `int` | Symbol expiration date |
| `trade_stops_level` | `int` | Minimum stop level in points |
| `trade_freeze_level` | `int` | Freeze level for trade operations |
| `trade_exemode` | `int` | Trade execution mode |
| `swap_mode` | `int` | Swap calculation mode |
| `swap_rollover3days` | `int` | Day of week for triple swap |
| `expiration_mode` | `int` | Allowed expiration modes (bitmask) |
| `filling_mode` | `int` | Allowed filling modes (bitmask) |
| `order_mode` | `int` | Allowed order types (bitmask) |
| `order_gtc_mode` | `int` | GTC order expiration mode |
| `option_mode` | `int` | Option type (0=EUROPEAN, 1=AMERICAN) |
| `option_right` | `int` | Option right (0=CALL, 1=PUT) |

**Decimal price fields**:
| Attribute | Type | Description |
|---|---|---|
| `bid` | `Decimal` | Current bid price |
| `bidhigh` | `Decimal` | Session high bid |
| `bidlow` | `Decimal` | Session low bid |
| `ask` | `Decimal` | Current ask price |
| `askhigh` | `Decimal` | Session high ask |
| `asklow` | `Decimal` | Session low ask |
| `last` | `Decimal` | Last deal price |
| `lasthigh` | `Decimal` | Session high last price |
| `lastlow` | `Decimal` | Session low last price |
| `point` | `Decimal` | Point size (minimum price change) |
| `option_strike` | `Decimal` | Option strike price |

**Decimal volume fields**:
| Attribute | Type | Description |
|---|---|---|
| `volume` | `Decimal` | Current volume |
| `volumehigh` | `Decimal` | Session high volume |
| `volumelow` | `Decimal` | Session low volume |
| `volume_real` | `Decimal` | Current real volume |
| `volumehigh_real` | `Decimal` | Session high real volume |
| `volumelow_real` | `Decimal` | Session low real volume |
| `volume_min` | `Decimal` | Minimum trade volume |
| `volume_max` | `Decimal` | Maximum trade volume |
| `volume_step` | `Decimal` | Volume change step |
| `volume_limit` | `Decimal` | Maximum allowed aggregate volume |

**Decimal trade/margin fields**:
| Attribute | Type | Description |
|---|---|---|
| `trade_tick_value` | `Decimal` | Value of a single tick |
| `trade_tick_value_profit` | `Decimal` | Tick value for profitable positions |
| `trade_tick_value_loss` | `Decimal` | Tick value for losing positions |
| `trade_tick_size` | `Decimal` | Minimum price change |
| `trade_contract_size` | `Decimal` | Contract size (lot size) |
| `trade_accrued_interest` | `Decimal` | Accrued interest |
| `trade_face_value` | `Decimal` | Face value |
| `trade_liquidity_rate` | `Decimal` | Liquidity rate |
| `swap_long` | `Decimal` | Long swap value |
| `swap_short` | `Decimal` | Short swap value |
| `margin_initial` | `Decimal` | Initial margin requirement |
| `margin_maintenance` | `Decimal` | Maintenance margin requirement |
| `margin_hedged` | `Decimal` | Margin for hedged positions |

**Decimal session fields**:
| Attribute | Type | Description |
|---|---|---|
| `session_volume` | `Decimal` | Session total volume |
| `session_turnover` | `Decimal` | Session turnover |
| `session_interest` | `Decimal` | Session open interest |
| `session_buy_orders_volume` | `Decimal` | Total volume of buy orders |
| `session_sell_orders_volume` | `Decimal` | Total volume of sell orders |
| `session_open` | `Decimal` | Session open price |
| `session_close` | `Decimal` | Session close price |
| `session_aw` | `Decimal` | Session average weighted price |
| `session_price_settlement` | `Decimal` | Settlement price |
| `session_price_limit_min` | `Decimal` | Minimum price limit |
| `session_price_limit_max` | `Decimal` | Maximum price limit |

**Decimal Greeks/price analysis**:
| Attribute | Type | Description |
|---|---|---|
| `price_change` | `Decimal` | Price change percentage |
| `price_volatility` | `Decimal` | Price volatility |
| `price_theoretical` | `Decimal` | Theoretical option price |
| `price_greeks_delta` | `Decimal` | Option delta |
| `price_greeks_theta` | `Decimal` | Option theta |
| `price_greeks_gamma` | `Decimal` | Option gamma |
| `price_greeks_vega` | `Decimal` | Option vega |
| `price_greeks_rho` | `Decimal` | Option rho |
| `price_greeks_omega` | `Decimal` | Option omega |
| `price_sensitivity` | `Decimal` | Price sensitivity |

**String metadata fields**:
| Attribute | Type | Description |
|---|---|---|
| `basis` | `str` | Underlying asset symbol |
| `category` | `str` | Symbol category |
| `currency_base` | `str` | Base currency (e.g., "EUR") |
| `currency_profit` | `str` | Profit currency |
| `currency_margin` | `str` | Margin currency |
| `bank` | `str` | Feeder bank |
| `description` | `str` | Symbol description text |
| `exchange` | `str` | Exchange name |
| `formula` | `str` | Custom symbol formula |
| `isin` | `str` | ISIN identifier |
| `name` | `str` | Symbol name (e.g., "EURUSD") |
| `page` | `str` | Information page URL |
| `path` | `str` | Symbol path in the symbol tree |

## Data Flow

```
YAML (default_symbols_info.yaml, ~30 FX pairs)
    |
    v
SharedData._load_default_symbols_info()  -- iterates YAML dict, creates SymbolInfo per symbol
    |
    v
SharedData.symbol_info[symbol_name] = SymbolInfo(...)
    |
    v
SymbolConnector.symbol_info(symbol) / symbols_get(group)
```

## Gaps & Issues

1. **No field defaults** -- All 97 fields are required. Constructing a `SymbolInfo` for testing requires providing every single field.
2. **No validators** -- No validation that `digits >= 0`, `volume_min <= volume_max`, `trade_mode` is a valid enum value, etc.
3. **`select` and `visible` are mutable** -- `SymbolConnector.symbol_select()` directly mutates these fields on the Pydantic model. In Pydantic v2, models are immutable by default unless `model_config` allows mutation.

## Requirements Derived

- **REQ-ENTITY-SYM-001**: `SymbolInfo` must contain all fields returned by `mt5.symbol_info()` to ensure full API compatibility.
- **REQ-ENTITY-SYM-002**: The `digits` field is critical for price reconstruction (`close / 10**digits`), so it must be accurately populated from symbol data.
- **REQ-ENTITY-SYM-003**: `select` and `visible` fields must be mutable to support `symbol_select()` operations.
