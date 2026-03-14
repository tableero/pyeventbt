# MT5 Responses Reference

Complete reference of all MetaTrader 5 response objects as used by PyEventBT. Each section documents the real MT5 Python API response shape, field types, realistic example values, and how PyEventBT maps them internally.

---

## Table of Contents

- [1. order\_send — OrderSendResult](#1-order_send--ordersendresult)
- [2. TradeRequest (nested in OrderSendResult)](#2-traderequest-nested-in-ordersendresult)
- [3. TradeDeal (from history\_deals\_get)](#3-tradedeal-from-history_deals_get)
- [4. TradePosition (from positions\_get)](#4-tradeposition-from-positions_get)
- [5. TradeOrder (from orders\_get)](#5-tradeorder-from-orders_get)
- [6. AccountInfo (from account\_info)](#6-accountinfo-from-account_info)
- [7. TerminalInfo (from terminal\_info)](#7-terminalinfo-from-terminal_info)
- [8. SymbolInfo (from symbol\_info)](#8-symbolinfo-from-symbol_info)
- [9. Tick (from symbol\_info\_tick)](#9-tick-from-symbol_info_tick)
- [10. copy\_rates\_from\_pos (bar data)](#10-copy_rates_from_pos-bar-data)
- [11. last\_error codes](#11-last_error-codes)
- [12. Trade Return Codes (retcode)](#12-trade-return-codes-retcode)
- [13. Enum Constants Reference](#13-enum-constants-reference)

---

## 1. order_send — OrderSendResult

Returned by `mt5.order_send(request)`. This is a named tuple in the real MT5 Python API; PyEventBT models it as a Pydantic `OrderSendResult`.

### Fields

| Field | Type | Description |
|---|---|---|
| `retcode` | `int` | Trade server return code (see [section 12](#12-trade-return-codes-retcode)) |
| `deal` | `int` | Deal ticket. **Known issue**: live accounts may return `0` here; use `result.order` with `history_deals_get(position=...)` instead |
| `order` | `int` | Order ticket — becomes the position `identifier`/`ticket` |
| `volume` | `float` | Executed volume |
| `price` | `float` | Execution price |
| `bid` | `float` | Current bid at execution time |
| `ask` | `float` | Current ask at execution time |
| `comment` | `str` | Broker comment on the result |
| `request_id` | `int` | Request ID set by the terminal |
| `retcode_external` | `int` | External trading system return code |
| `request` | `TradeRequest` | The original request that produced this result |

### Example: Successful Market Order (BUY)

```python
# Real MT5 response from mt5.order_send()
OrderSendResult(
    retcode=10009,              # TRADE_RETCODE_DONE
    deal=123456789,             # Deal ticket (may be 0 on live — see known issue)
    order=987654321,            # Order ticket
    volume=0.1,
    price=1.08542,
    bid=1.08540,
    ask=1.08542,
    comment="Request executed",
    request_id=3294967295,
    retcode_external=0,
    request=TradeRequest(
        action=1,               # TRADE_ACTION_DEAL
        magic=12345,
        order=0,
        symbol="EURUSD",
        volume=0.1,
        price=0.0,              # 0 for market orders
        stoplimit=0.0,
        sl=1.08200,
        tp=1.09000,
        deviation=0,
        type=0,                 # ORDER_TYPE_BUY
        type_filling=0,         # ORDER_FILLING_FOK
        type_time=0,            # ORDER_TIME_GTC
        expiration=0,
        comment="12345-MKT",
        position=0,
        position_by=0
    )
)
```

### Example: Successful Market Order (SELL)

```python
OrderSendResult(
    retcode=10009,
    deal=123456790,
    order=987654322,
    volume=0.05,
    price=1.08540,              # Sells fill at bid
    bid=1.08540,
    ask=1.08542,
    comment="Request executed",
    request_id=3294967296,
    retcode_external=0,
    request=TradeRequest(
        action=1,               # TRADE_ACTION_DEAL
        magic=12345,
        order=0,
        symbol="EURUSD",
        volume=0.05,
        price=0.0,
        stoplimit=0.0,
        sl=1.08900,
        tp=1.08100,
        deviation=0,
        type=1,                 # ORDER_TYPE_SELL
        type_filling=0,
        type_time=0,
        expiration=0,
        comment="12345-MKT",
        position=0,
        position_by=0
    )
)
```

### Example: Successful Pending Order (BUY LIMIT)

```python
OrderSendResult(
    retcode=10009,
    deal=0,                     # No deal yet — order is pending
    order=987654400,
    volume=0.1,
    price=1.08000,              # The limit price
    bid=1.08540,
    ask=1.08542,
    comment="Request executed",
    request_id=3294967297,
    retcode_external=0,
    request=TradeRequest(
        action=5,               # TRADE_ACTION_PENDING
        magic=12345,
        order=0,
        symbol="EURUSD",
        volume=0.1,
        price=1.08000,
        stoplimit=0.0,
        sl=1.07800,
        tp=1.08500,
        deviation=0,
        type=2,                 # ORDER_TYPE_BUY_LIMIT
        type_filling=0,
        type_time=0,            # ORDER_TIME_GTC
        expiration=0,
        comment="12345-PDG",
        position=0,
        position_by=0
    )
)
```

### Example: Successful Pending Order (SELL STOP)

```python
OrderSendResult(
    retcode=10009,
    deal=0,
    order=987654401,
    volume=0.2,
    price=1.08000,
    bid=1.08540,
    ask=1.08542,
    comment="Request executed",
    request_id=3294967298,
    retcode_external=0,
    request=TradeRequest(
        action=5,               # TRADE_ACTION_PENDING
        magic=12345,
        order=0,
        symbol="EURUSD",
        volume=0.2,
        price=1.08000,
        stoplimit=0.0,
        sl=1.08300,
        tp=1.07500,
        deviation=0,
        type=5,                 # ORDER_TYPE_SELL_STOP
        type_filling=0,
        type_time=0,
        expiration=0,
        comment="12345-PDG",
        position=0,
        position_by=0
    )
)
```

### Example: Close Position

```python
OrderSendResult(
    retcode=10009,
    deal=123456800,
    order=987654321,            # Same as the position ticket
    volume=0.1,
    price=1.08650,
    bid=1.08650,
    ask=1.08652,
    comment="Request executed",
    request_id=3294967299,
    retcode_external=0,
    request=TradeRequest(
        action=1,               # TRADE_ACTION_DEAL
        magic=12345,
        order=0,
        symbol="EURUSD",
        volume=0.1,
        price=0.0,
        stoplimit=0.0,
        sl=0.0,
        tp=0.0,
        deviation=0,
        type=1,                 # ORDER_TYPE_SELL (opposite of BUY position)
        type_filling=0,
        type_time=0,
        expiration=0,
        comment="12345-Close position",
        position=987654321,     # The position ticket to close
        position_by=0
    )
)
```

### Example: Modify SL/TP

```python
OrderSendResult(
    retcode=10009,
    deal=0,
    order=0,
    volume=0.0,
    price=0.0,
    bid=0.0,
    ask=0.0,
    comment="Request executed",
    request_id=3294967300,
    retcode_external=0,
    request=TradeRequest(
        action=6,               # TRADE_ACTION_SLTP
        magic=0,
        order=0,
        symbol="",
        volume=0.0,
        price=0.0,
        stoplimit=0.0,
        sl=1.08300,
        tp=1.09100,
        deviation=0,
        type=0,
        type_filling=0,
        type_time=0,
        expiration=0,
        comment="",
        position=987654321,     # The position to modify
        position_by=0
    )
)
```

### Example: Cancel Pending Order

```python
OrderSendResult(
    retcode=10009,
    deal=0,
    order=987654400,            # The order being cancelled
    volume=0.0,
    price=0.0,
    bid=0.0,
    ask=0.0,
    comment="Request executed",
    request_id=3294967301,
    retcode_external=0,
    request=TradeRequest(
        action=8,               # TRADE_ACTION_REMOVE
        magic=0,
        order=987654400,
        symbol="",
        volume=0.0,
        price=0.0,
        stoplimit=0.0,
        sl=0.0,
        tp=0.0,
        deviation=0,
        type=0,
        type_filling=0,
        type_time=0,
        expiration=0,
        comment="",
        position=0,
        position_by=0
    )
)
```

### Example: Failed — Not Enough Money

```python
OrderSendResult(
    retcode=10019,              # TRADE_RETCODE_NO_MONEY
    deal=0,
    order=0,
    volume=0.0,
    price=0.0,
    bid=1.08540,
    ask=1.08542,
    comment="No money",
    request_id=3294967302,
    retcode_external=0,
    request=TradeRequest(...)   # The original request
)
```

### Example: Failed — Invalid Stops

```python
OrderSendResult(
    retcode=10016,              # TRADE_RETCODE_INVALID_STOPS
    deal=0,
    order=0,
    volume=0.0,
    price=0.0,
    bid=1.08540,
    ask=1.08542,
    comment="Invalid stops",
    request_id=3294967303,
    retcode_external=0,
    request=TradeRequest(...)
)
```

### Example: Failed — Market Closed

```python
OrderSendResult(
    retcode=10018,              # TRADE_RETCODE_MARKET_CLOSED
    deal=0,
    order=0,
    volume=0.0,
    price=0.0,
    bid=0.0,
    ask=0.0,
    comment="Market is closed",
    request_id=3294967304,
    retcode_external=0,
    request=TradeRequest(...)
)
```

### Example: Failed — Invalid Volume

```python
OrderSendResult(
    retcode=10014,              # TRADE_RETCODE_INVALID_VOLUME
    deal=0,
    order=0,
    volume=0.0,
    price=0.0,
    bid=1.08540,
    ask=1.08542,
    comment="Invalid volume",
    request_id=3294967305,
    retcode_external=0,
    request=TradeRequest(...)
)
```

### Example: Failed — Requote

```python
OrderSendResult(
    retcode=10004,              # TRADE_RETCODE_REQUOTE
    deal=0,
    order=0,
    volume=0.0,
    price=1.08550,              # New requoted price
    bid=1.08548,
    ask=1.08550,
    comment="Requote",
    request_id=3294967306,
    retcode_external=0,
    request=TradeRequest(...)
)
```

### Known Issues (Live)

- `deal` field returns `0` on some live brokers even for successful market orders. Workaround: use `mt5.history_deals_get(position=result.order)` to retrieve the actual deal(s).
- Deals may not be immediately available after `order_send` returns. PyEventBT retries up to 100 times (5 seconds) with 50ms sleep between attempts.

---

## 2. TradeRequest (nested in OrderSendResult)

The trade request that was sent to the server. Accessed via `result.request` on the real MT5 API (named tuple), or `result.request._asdict()` to convert to dict.

### Fields

| Field | Type | Description |
|---|---|---|
| `action` | `int` | Trade action type (see [TRADE_ACTION constants](#trade-actions)) |
| `magic` | `int` | Expert Advisor ID (strategy_id in PyEventBT) |
| `order` | `int` | Order ticket (for modify/cancel operations) |
| `symbol` | `str` | Trading symbol |
| `volume` | `float` | Requested volume in lots |
| `price` | `float` | Price (0.0 for market orders) |
| `stoplimit` | `float` | StopLimit level for ORDER_TYPE_BUY_STOP_LIMIT / SELL_STOP_LIMIT |
| `sl` | `float` | Stop Loss level |
| `tp` | `float` | Take Profit level |
| `deviation` | `int` | Maximum price deviation (slippage) in points |
| `type` | `int` | Order type (see [ORDER_TYPE constants](#order-types)) |
| `type_filling` | `int` | Order filling type (see [ORDER_FILLING constants](#order-filling-types)) |
| `type_time` | `int` | Order lifetime type (see [ORDER_TIME constants](#order-time-types)) |
| `expiration` | `int` | Order expiration time (unix timestamp, 0 = no expiration) |
| `comment` | `str` | Order comment (max 31 characters in MT5) |
| `position` | `int` | Position ticket (for close/modify operations) |
| `position_by` | `int` | Opposite position ticket (for close-by operations) |

---

## 3. TradeDeal (from history_deals_get)

Returned by `mt5.history_deals_get(position=...) ` or `mt5.history_deals_get(ticket=...)`. Each deal represents a single execution within an order.

### Fields

| Field | Type | Description |
|---|---|---|
| `ticket` | `int` | Unique deal ticket |
| `order` | `int` | Order ticket that triggered this deal |
| `time` | `int` | Deal execution time (unix seconds) |
| `time_msc` | `int` | Deal execution time (unix milliseconds) |
| `type` | `int` | Deal type: `0`=buy, `1`=sell, `2`=balance, `3`=credit |
| `entry` | `int` | Deal entry: `0`=in (open), `1`=out (close), `2`=reverse, `3`=close-by |
| `magic` | `int` | Expert Advisor ID |
| `position_id` | `int` | Position ID that the deal belongs to |
| `reason` | `int` | Deal reason (see [DEAL_REASON constants](#deal-reasons)) |
| `volume` | `float` | Deal volume in lots |
| `price` | `float` | Deal execution price |
| `commission` | `float` | Commission (negative value = cost charged) |
| `swap` | `float` | Swap value |
| `profit` | `float` | Profit in deposit currency. Always `0.0` for entry deals (entry=0) |
| `fee` | `float` | Fee |
| `symbol` | `str` | Symbol |
| `comment` | `str` | Deal comment |
| `external_id` | `str` | Deal ID in external system |

### Example: Entry Deal (Opening a BUY position)

```python
TradeDeal(
    ticket=500001,
    order=987654321,
    time=1710400200,            # 2024-03-14 10:30:00 UTC
    time_msc=1710400200000,
    type=0,                     # DEAL_TYPE_BUY
    entry=0,                    # DEAL_ENTRY_IN
    magic=12345,
    position_id=987654321,
    reason=3,                   # DEAL_REASON_EXPERT
    volume=0.1,
    price=1.08542,
    commission=-0.70,           # Commission is negative (charged)
    swap=0.0,
    profit=0.0,                 # Always 0 for entry deals
    fee=0.0,
    symbol="EURUSD",
    comment="12345-MKT",
    external_id=""
)
```

### Example: Exit Deal (Closing a BUY position)

```python
TradeDeal(
    ticket=500002,
    order=987654322,
    time=1710486600,            # 2024-03-15 10:30:00 UTC
    time_msc=1710486600000,
    type=1,                     # DEAL_TYPE_SELL (closing a BUY)
    entry=1,                    # DEAL_ENTRY_OUT
    magic=12345,
    position_id=987654321,      # Same position_id as the entry deal
    reason=3,                   # DEAL_REASON_EXPERT
    volume=0.1,
    price=1.08750,
    commission=-0.70,
    swap=-1.25,                 # Overnight swap charged
    profit=20.80,               # Profit in deposit currency
    fee=0.0,
    symbol="EURUSD",
    comment="12345-Close position",
    external_id=""
)
```

### Important Notes

- One order can produce multiple deals (partial fills).
- `profit` is `0.0` for all `entry=0` (IN) deals. The P&L only appears on `entry=1` (OUT) deals.
- `commission` is typically negative (a charge). PyEventBT uses `abs(deal.commission)` when creating `FillEvent`.

---

## 4. TradePosition (from positions_get)

Returned by `mt5.positions_get()`, `mt5.positions_get(symbol=...)`, or `mt5.positions_get(ticket=...)`. Represents a currently open position.

### Fields

| Field | Type | Description |
|---|---|---|
| `ticket` | `int` | Position ticket (unique identifier) |
| `time` | `int` | Open time (unix seconds) |
| `time_msc` | `int` | Open time (unix milliseconds) |
| `time_update` | `int` | Last update time (unix seconds) |
| `time_update_msc` | `int` | Last update time (unix milliseconds) |
| `type` | `int` | Position type: `0`=buy, `1`=sell |
| `magic` | `int` | Expert Advisor ID |
| `identifier` | `int` | Position identifier (same as `ticket` for hedging accounts) |
| `reason` | `int` | Position open reason |
| `volume` | `float` | Current position volume |
| `price_open` | `float` | Position open price |
| `sl` | `float` | Stop Loss level |
| `tp` | `float` | Take Profit level |
| `price_current` | `float` | Current price of the symbol |
| `swap` | `float` | Accumulated swap |
| `profit` | `float` | Current floating profit |
| `symbol` | `str` | Symbol |
| `comment` | `str` | Comment |
| `external_id` | `str` | External system position ID |

### Example: Open BUY Position

```python
TradePosition(
    ticket=987654321,
    time=1710400200,
    time_msc=1710400200000,
    time_update=1710400200,
    time_update_msc=1710400200000,
    type=0,                     # BUY
    magic=12345,
    identifier=987654321,
    reason=3,                   # DEAL_REASON_EXPERT
    volume=0.1,
    price_open=1.08542,
    sl=1.08200,
    tp=1.09000,
    price_current=1.08650,
    swap=-1.25,
    profit=10.80,               # Floating unrealized P&L
    symbol="EURUSD",
    comment="12345-MKT",
    external_id=""
)
```

### Example: Open SELL Position

```python
TradePosition(
    ticket=987654322,
    time=1710403800,
    time_msc=1710403800000,
    time_update=1710403800,
    time_update_msc=1710403800000,
    type=1,                     # SELL
    magic=12345,
    identifier=987654322,
    reason=3,
    volume=0.05,
    price_open=1.08700,
    sl=1.08900,
    tp=1.08200,
    price_current=1.08650,
    swap=0.0,
    profit=2.50,
    symbol="EURUSD",
    comment="12345-MKT",
    external_id=""
)
```

---

## 5. TradeOrder (from orders_get)

Returned by `mt5.orders_get()`. Represents a currently active **pending** order (not yet executed).

### Fields

| Field | Type | Description |
|---|---|---|
| `ticket` | `int` | Order ticket |
| `time_setup` | `int` | Order placement time (unix seconds) |
| `time_setup_msc` | `int` | Order placement time (unix milliseconds) |
| `time_done` | `int` | Order execution/cancellation time |
| `time_done_msc` | `int` | Same in milliseconds |
| `time_expiration` | `int` | Expiration time (0 = no expiration) |
| `type` | `int` | Order type (see [ORDER_TYPE constants](#order-types)) |
| `type_time` | `int` | Order lifetime type |
| `type_filling` | `int` | Order filling type |
| `state` | `int` | Order state (see [ORDER_STATE constants](#order-states)) |
| `magic` | `int` | Expert Advisor ID |
| `position_id` | `int` | Position ID (if linked to a position) |
| `position_by_id` | `int` | Opposite position ID |
| `reason` | `int` | Order placement reason |
| `volume_initial` | `float` | Initial order volume |
| `volume_current` | `float` | Current (unfilled) volume |
| `price_open` | `float` | Order price |
| `sl` | `float` | Stop Loss |
| `tp` | `float` | Take Profit |
| `price_current` | `float` | Current symbol price |
| `price_stoplimit` | `float` | StopLimit price |
| `symbol` | `str` | Symbol |
| `comment` | `str` | Comment |
| `external_id` | `str` | External system ID |

### Example: Active BUY LIMIT Order

```python
TradeOrder(
    ticket=987654400,
    time_setup=1710400200,
    time_setup_msc=1710400200000,
    time_done=0,                # Not yet executed
    time_done_msc=0,
    time_expiration=0,          # GTC
    type=2,                     # ORDER_TYPE_BUY_LIMIT
    type_time=0,                # ORDER_TIME_GTC
    type_filling=0,             # ORDER_FILLING_FOK
    state=1,                    # ORDER_STATE_PLACED
    magic=12345,
    position_id=0,
    position_by_id=0,
    reason=3,                   # ORDER_REASON_EXPERT
    volume_initial=0.1,
    volume_current=0.1,         # No partial fills yet
    price_open=1.08000,
    sl=1.07800,
    tp=1.08500,
    price_current=1.08540,
    price_stoplimit=0.0,
    symbol="EURUSD",
    comment="12345-PDG",
    external_id=""
)
```

---

## 6. AccountInfo (from account_info)

Returned by `mt5.account_info()`. Use `._asdict()` to convert to a dictionary.

### Fields

| Field | Type | Description |
|---|---|---|
| `login` | `int` | Account number |
| `trade_mode` | `int` | `0`=demo, `1`=contest, `2`=real |
| `leverage` | `int` | Account leverage (e.g. 100, 500) |
| `limit_orders` | `int` | Maximum number of active pending orders |
| `margin_so_mode` | `int` | Stop Out mode: `0`=percent, `1`=money |
| `trade_allowed` | `bool` | Is trading allowed |
| `trade_expert` | `bool` | Is automated trading allowed |
| `margin_mode` | `int` | `0`=netting, `1`=exchange, `2`=hedging |
| `currency_digits` | `int` | Decimal digits for the account currency |
| `fifo_close` | `bool` | FIFO close required |
| `balance` | `float` | Account balance |
| `credit` | `float` | Account credit |
| `profit` | `float` | Current floating profit |
| `equity` | `float` | Account equity (balance + credit + profit) |
| `margin` | `float` | Used margin |
| `margin_free` | `float` | Free margin |
| `margin_level` | `float` | Margin level (%) |
| `margin_so_call` | `float` | Margin call level |
| `margin_so_so` | `float` | Stop out level |
| `margin_initial` | `float` | Initial margin required |
| `margin_maintenance` | `float` | Maintenance margin |
| `assets` | `float` | Current assets |
| `liabilities` | `float` | Current liabilities |
| `commission_blocked` | `float` | Blocked commission |
| `name` | `str` | Client name |
| `server` | `str` | Server name |
| `currency` | `str` | Account currency (e.g. "USD", "EUR") |
| `company` | `str` | Broker company name |

### Example

```python
AccountInfo(
    login=12345678,
    trade_mode=0,               # ACCOUNT_TRADE_MODE_DEMO
    leverage=100,
    limit_orders=200,
    margin_so_mode=0,           # ACCOUNT_STOPOUT_MODE_PERCENT
    trade_allowed=True,
    trade_expert=True,
    margin_mode=2,              # ACCOUNT_MARGIN_MODE_RETAIL_HEDGING
    currency_digits=2,
    fifo_close=False,
    balance=10000.00,
    credit=0.0,
    profit=15.30,
    equity=10015.30,
    margin=108.54,
    margin_free=9906.76,
    margin_level=9228.34,
    margin_so_call=50.0,
    margin_so_so=30.0,
    margin_initial=0.0,
    margin_maintenance=0.0,
    assets=0.0,
    liabilities=0.0,
    commission_blocked=0.0,
    name="John Doe",
    server="MetaQuotes-Demo",
    currency="USD",
    company="MetaQuotes Software Corp."
)
```

---

## 7. TerminalInfo (from terminal_info)

Returned by `mt5.terminal_info()`.

### Fields

| Field | Type | Description |
|---|---|---|
| `community_account` | `bool` | Is MQL5.community account authorized |
| `community_connection` | `bool` | Is connection to MQL5.community established |
| `connected` | `bool` | Is terminal connected to the trade server |
| `dlls_allowed` | `bool` | Is DLL usage allowed |
| `trade_allowed` | `bool` | Is trading allowed (algo trading button) |
| `tradeapi_disabled` | `bool` | Is trade API disabled |
| `email_enabled` | `bool` | Is email sending enabled |
| `ftp_enabled` | `bool` | Is FTP publishing enabled |
| `notifications_enabled` | `bool` | Is push notifications enabled |
| `mqid` | `bool` | Is MetaQuotes ID set |
| `build` | `int` | Terminal build number |
| `maxbars` | `int` | Maximum bars in chart |
| `codepage` | `int` | Terminal code page |
| `ping_last` | `int` | Last known ping to server (microseconds) |
| `community_balance` | `float` | MQL5.community balance |
| `retransmission` | `float` | Network retransmission rate |
| `company` | `str` | Broker company |
| `name` | `str` | Terminal name |
| `language` | `str` | Terminal language |
| `path` | `str` | Terminal installation path |
| `data_path` | `str` | Terminal data path |
| `commondata_path` | `str` | Common data path |

### Example

```python
TerminalInfo(
    community_account=False,
    community_connection=False,
    connected=True,
    dlls_allowed=False,
    trade_allowed=True,
    tradeapi_disabled=False,
    email_enabled=False,
    ftp_enabled=False,
    notifications_enabled=False,
    mqid=False,
    build=4150,
    maxbars=100000,
    codepage=0,
    ping_last=52340,            # ~52ms
    community_balance=0.0,
    retransmission=0.0,
    company="MetaQuotes Software Corp.",
    name="MetaTrader 5",
    language="English",
    path="C:\\Program Files\\MetaTrader 5",
    data_path="C:\\Users\\user\\AppData\\Roaming\\MetaQuotes\\Terminal\\ABC123",
    commondata_path="C:\\Users\\user\\AppData\\Roaming\\MetaQuotes\\Terminal\\Common"
)
```

---

## 8. SymbolInfo (from symbol_info)

Returned by `mt5.symbol_info(symbol)`. Extensive — key trading fields highlighted below.

### Key Trading Fields

| Field | Type | Description |
|---|---|---|
| `name` | `str` | Symbol name |
| `visible` | `bool` | Is symbol visible in Market Watch |
| `select` | `bool` | Is symbol selected in Market Watch |
| `digits` | `int` | Price decimal places (e.g. 5 for EURUSD) |
| `spread` | `int` | Current spread in points |
| `spread_float` | `bool` | Is spread floating |
| `bid` | `float` | Current bid price |
| `ask` | `float` | Current ask price |
| `point` | `float` | Point value (e.g. 0.00001 for 5-digit EURUSD) |
| `trade_tick_value` | `float` | Tick value in deposit currency |
| `trade_tick_size` | `float` | Minimum price change |
| `trade_contract_size` | `float` | Contract size (e.g. 100000 for Forex) |
| `volume_min` | `float` | Minimum volume (e.g. 0.01) |
| `volume_max` | `float` | Maximum volume (e.g. 100.0) |
| `volume_step` | `float` | Volume step (e.g. 0.01) |
| `trade_mode` | `int` | Trading mode (see [SYMBOL_TRADE_MODE](#symbol-trade-modes)) |
| `trade_calc_mode` | `int` | Margin calculation mode (0=Forex, 2=CFD, etc.) |
| `swap_long` | `float` | Long swap rate |
| `swap_short` | `float` | Short swap rate |
| `swap_mode` | `int` | Swap calculation mode |
| `currency_base` | `str` | Base currency (e.g. "EUR") |
| `currency_profit` | `str` | Profit currency (e.g. "USD") |
| `currency_margin` | `str` | Margin currency (e.g. "EUR") |

### Example: EURUSD

```python
SymbolInfo(
    custom=False,
    chart_mode=0,
    select=True,
    visible=True,
    session_deals=0,
    session_buy_orders=0,
    session_sell_orders=0,
    volume=0,
    volumehigh=0,
    volumelow=0,
    time=1710486600,
    digits=5,
    spread=12,                  # 1.2 pips
    spread_float=True,
    ticks_bookdepth=10,
    trade_calc_mode=0,          # SYMBOL_CALC_MODE_FOREX
    trade_mode=4,               # SYMBOL_TRADE_MODE_FULL
    start_time=0,
    expiration_time=0,
    trade_stops_level=0,
    trade_freeze_level=0,
    trade_exemode=2,            # SYMBOL_TRADE_EXECUTION_MARKET
    swap_mode=1,                # SYMBOL_SWAP_MODE_POINTS
    swap_rollover3days=3,       # Wednesday
    margin_hedged_use_leg=False,
    expiration_mode=15,
    filling_mode=1,
    order_mode=127,
    order_gtc_mode=0,
    option_mode=0,
    option_right=0,
    bid=1.08540,
    bidhigh=1.08900,
    bidlow=1.08200,
    ask=1.08552,
    askhigh=1.08912,
    asklow=1.08212,
    last=0.0,
    lasthigh=0.0,
    lastlow=0.0,
    volume_real=0.0,
    volumehigh_real=0.0,
    volumelow_real=0.0,
    option_strike=0.0,
    point=0.00001,
    trade_tick_value=1.0,       # 1 tick = $1 per lot
    trade_tick_value_profit=1.0,
    trade_tick_value_loss=1.0,
    trade_tick_size=0.00001,
    trade_contract_size=100000.0,
    trade_accrued_interest=0.0,
    trade_face_value=0.0,
    trade_liquidity_rate=0.0,
    volume_min=0.01,
    volume_max=100.0,
    volume_step=0.01,
    volume_limit=0.0,
    swap_long=-6.3,
    swap_short=1.2,
    margin_initial=0.0,
    margin_maintenance=0.0,
    session_volume=0.0,
    session_turnover=0.0,
    session_interest=0.0,
    session_buy_orders_volume=0.0,
    session_sell_orders_volume=0.0,
    session_open=1.08450,
    session_close=1.08350,
    session_aw=0.0,
    session_price_settlement=0.0,
    session_price_limit_min=0.0,
    session_price_limit_max=0.0,
    margin_hedged=100000.0,
    price_change=0.17,
    price_volatility=0.0,
    price_theoretical=0.0,
    price_greeks_delta=0.0,
    price_greeks_theta=0.0,
    price_greeks_gamma=0.0,
    price_greeks_vega=0.0,
    price_greeks_rho=0.0,
    price_greeks_omega=0.0,
    price_sensitivity=0.0,
    basis="",
    category="",
    currency_base="EUR",
    currency_profit="USD",
    currency_margin="EUR",
    bank="",
    description="Euro vs US Dollar",
    exchange="",
    formula="",
    isin="",
    name="EURUSD",
    page="",
    path="Forex\\EURUSD"
)
```

---

## 9. Tick (from symbol_info_tick)

Returned by `mt5.symbol_info_tick(symbol)`.

### Fields

| Field | Type | Description |
|---|---|---|
| `time` | `int` | Tick time (unix seconds) |
| `bid` | `float` | Bid price |
| `ask` | `float` | Ask price |
| `last` | `float` | Last deal price |
| `volume` | `int` | Volume for the last deal |
| `time_msc` | `int` | Tick time (unix milliseconds) |
| `flags` | `int` | Tick flags (bitmask, see [TICK_FLAG constants](#tick-flags)) |
| `volume_real` | `float` | Real volume for the last deal |

### Example

```python
Tick(
    time=1710486600,
    bid=1.08540,
    ask=1.08552,
    last=0.0,                   # 0 for Forex (no last price)
    volume=0,
    time_msc=1710486600123,
    flags=6,                    # TICK_FLAG_BID | TICK_FLAG_ASK
    volume_real=0.0
)
```

### PyEventBT Conversion

The live data provider converts ticks to a dictionary:

```python
{
    'time': 1710486600,
    'bid': Decimal('1.08540'),
    'ask': Decimal('1.08552'),
    'last': Decimal('0.0'),
    'volume': 0,
    'time_msc': 1710486600123,
    'flags': 6,
    'volume_real': Decimal('0.0')
}
```

---

## 10. copy_rates_from_pos (bar data)

Returned by `mt5.copy_rates_from_pos(symbol, timeframe, start_pos, count)` as a numpy structured array. Each element has these fields:

### Fields

| Field | Type | Description |
|---|---|---|
| `time` | `int` | Bar open time (unix seconds, server time) |
| `open` | `float` | Open price |
| `high` | `float` | High price |
| `low` | `float` | Low price |
| `close` | `float` | Close price |
| `tick_volume` | `int` | Tick volume |
| `spread` | `int` | Spread in points |
| `real_volume` | `int` | Real volume |

### Example: Raw MT5 response

```python
# mt5.copy_rates_from_pos("EURUSD", mt5.TIMEFRAME_M1, 1, 3) returns:
[
    (1710486000, 1.08530, 1.08545, 1.08520, 1.08540, 145, 12, 0),
    (1710486060, 1.08540, 1.08560, 1.08535, 1.08550, 98,  12, 0),
    (1710486120, 1.08550, 1.08555, 1.08530, 1.08535, 112, 13, 0),
]
```

### PyEventBT Conversion to BarEvent

Prices are stored as integers multiplied by `10^digits`:

```python
BarEvent(
    type="BAR",
    symbol="EURUSD",
    datetime=datetime(2024, 3, 15, 10, 0, 0, tzinfo=timezone.utc),
    data=Bar(
        open=108530,            # 1.08530 * 10^5
        high=108545,
        low=108520,
        close=108540,
        tickvol=145,
        volume=0,
        spread=12,
        digits=5
    ),
    timeframe="1min"
)
```

### PyEventBT Conversion to Polars DataFrame

```python
# get_latest_bars("EURUSD", "1min", N=3)
shape: (3, 8)
+---------------------+---------+---------+---------+---------+---------+--------+--------+
| datetime            | open    | high    | low     | close   | tickvol | volume | spread |
| datetime[ms]        | f64     | f64     | f64     | f64     | i64     | i64    | i64    |
+---------------------+---------+---------+---------+---------+---------+--------+--------+
| 2024-03-15 10:00:00 | 1.08530 | 1.08545 | 1.08520 | 1.08540 | 145     | 0      | 12     |
| 2024-03-15 10:01:00 | 1.08540 | 1.08560 | 1.08535 | 1.08550 | 98      | 0      | 12     |
| 2024-03-15 10:02:00 | 1.08550 | 1.08555 | 1.08530 | 1.08535 | 112     | 0      | 13     |
+---------------------+---------+---------+---------+---------+---------+--------+--------+
```

---

## 11. last_error codes

Returned by `mt5.last_error()` as a `(code, message)` tuple.

| Code | Constant | Description |
|---|---|---|
| `1` | `RES_S_OK` | Success |
| `-1` | `RES_E_FAIL` | Generic fail |
| `-2` | `RES_E_INVALID_PARAMS` | Invalid arguments/parameters |
| `-3` | `RES_E_NO_MEMORY` | No memory condition |
| `-4` | `RES_E_NOT_FOUND` | No history / not found |
| `-5` | `RES_E_INVALID_VERSION` | Invalid version |
| `-6` | `RES_E_AUTH_FAILED` | Authorization failed |
| `-7` | `RES_E_UNSUPPORTED` | Unsupported method |
| `-8` | `RES_E_AUTO_TRADING_DISABLED` | Auto-trading disabled |
| `-10000` | `RES_E_INTERNAL_FAIL` | Internal IPC general error |
| `-10001` | `RES_E_INTERNAL_FAIL_SEND` | Internal IPC send failed |
| `-10002` | `RES_E_INTERNAL_FAIL_RECEIVE` | Internal IPC recv failed |
| `-10003` | `RES_E_INTERNAL_FAIL_INIT` | Internal IPC initialization fail |
| `-10004` | `RES_E_INTERNAL_FAIL_CONNECT` | Internal IPC no connection |
| `-10005` | `RES_E_INTERNAL_FAIL_TIMEOUT` | Internal timeout |

### Example

```python
mt5.last_error()
# Success:  (1, 'Success')
# Failed:   (-6, 'Terminal: Authorization failed')
# Not found: (-4, 'Terminal: Not found')
```

---

## 12. Trade Return Codes (retcode)

The `retcode` field in `OrderSendResult`. Only `10009` (DONE) and `10025` (NO_CHANGES) are treated as success by PyEventBT.

| Code | Constant | Description |
|---|---|---|
| `10004` | `TRADE_RETCODE_REQUOTE` | Requote — price has changed |
| `10006` | `TRADE_RETCODE_REJECT` | Request rejected |
| `10007` | `TRADE_RETCODE_CANCEL` | Request canceled by trader |
| `10008` | `TRADE_RETCODE_PLACED` | Order placed |
| **`10009`** | **`TRADE_RETCODE_DONE`** | **Request completed (success)** |
| `10010` | `TRADE_RETCODE_DONE_PARTIAL` | Only part of the request was completed |
| `10011` | `TRADE_RETCODE_ERROR` | Request processing error |
| `10012` | `TRADE_RETCODE_TIMEOUT` | Request timeout |
| `10013` | `TRADE_RETCODE_INVALID` | Invalid request |
| `10014` | `TRADE_RETCODE_INVALID_VOLUME` | Invalid volume in the request |
| `10015` | `TRADE_RETCODE_INVALID_PRICE` | Invalid price in the request |
| `10016` | `TRADE_RETCODE_INVALID_STOPS` | Invalid stops (SL/TP) |
| `10017` | `TRADE_RETCODE_TRADE_DISABLED` | Trade is disabled |
| `10018` | `TRADE_RETCODE_MARKET_CLOSED` | Market is closed |
| `10019` | `TRADE_RETCODE_NO_MONEY` | Insufficient funds |
| `10020` | `TRADE_RETCODE_PRICE_CHANGED` | Price has changed |
| `10021` | `TRADE_RETCODE_PRICE_OFF` | No quotes to process the request |
| `10022` | `TRADE_RETCODE_INVALID_EXPIRATION` | Invalid order expiration date |
| `10023` | `TRADE_RETCODE_ORDER_CHANGED` | Order state changed |
| `10024` | `TRADE_RETCODE_TOO_MANY_REQUESTS` | Too frequent requests |
| **`10025`** | **`TRADE_RETCODE_NO_CHANGES`** | **No changes in request (success)** |
| `10026` | `TRADE_RETCODE_SERVER_DISABLES_AT` | Autotrading disabled by server |
| `10027` | `TRADE_RETCODE_CLIENT_DISABLES_AT` | Autotrading disabled by client |
| `10028` | `TRADE_RETCODE_LOCKED` | Request locked for processing |
| `10029` | `TRADE_RETCODE_FROZEN` | Order/position frozen |
| `10030` | `TRADE_RETCODE_INVALID_FILL` | Invalid fill type |
| `10031` | `TRADE_RETCODE_CONNECTION` | No connection with trade server |
| `10032` | `TRADE_RETCODE_ONLY_REAL` | Operation allowed only for real accounts |
| `10033` | `TRADE_RETCODE_LIMIT_ORDERS` | Pending orders limit reached |
| `10034` | `TRADE_RETCODE_LIMIT_VOLUME` | Volume limit for symbol reached |
| `10035` | `TRADE_RETCODE_INVALID_ORDER` | Incorrect or prohibited order type |
| `10036` | `TRADE_RETCODE_POSITION_CLOSED` | Position already closed |
| `10038` | `TRADE_RETCODE_INVALID_CLOSE_VOLUME` | Close volume exceeds position volume |
| `10039` | `TRADE_RETCODE_CLOSE_ORDER_EXIST` | Close order already exists |
| `10040` | `TRADE_RETCODE_LIMIT_POSITIONS` | Maximum positions limit reached |
| `10041` | `TRADE_RETCODE_REJECT_CANCEL` | Request to cancel pending order rejected |
| `10042` | `TRADE_RETCODE_LONG_ONLY` | Only long positions allowed |
| `10043` | `TRADE_RETCODE_SHORT_ONLY` | Only short positions allowed |
| `10044` | `TRADE_RETCODE_CLOSE_ONLY` | Only position close allowed |
| `10045` | `TRADE_RETCODE_FIFO_CLOSE` | FIFO rule: can only close oldest position |

---

## 13. Enum Constants Reference

### Trade Actions

| Value | Constant | Description |
|---|---|---|
| `1` | `TRADE_ACTION_DEAL` | Market order |
| `5` | `TRADE_ACTION_PENDING` | Pending order |
| `6` | `TRADE_ACTION_SLTP` | Modify SL/TP |
| `7` | `TRADE_ACTION_MODIFY` | Modify pending order |
| `8` | `TRADE_ACTION_REMOVE` | Delete pending order |
| `10` | `TRADE_ACTION_CLOSE_BY` | Close by opposite position |

### Order Types

| Value | Constant | Description |
|---|---|---|
| `0` | `ORDER_TYPE_BUY` | Market buy |
| `1` | `ORDER_TYPE_SELL` | Market sell |
| `2` | `ORDER_TYPE_BUY_LIMIT` | Buy Limit |
| `3` | `ORDER_TYPE_SELL_LIMIT` | Sell Limit |
| `4` | `ORDER_TYPE_BUY_STOP` | Buy Stop |
| `5` | `ORDER_TYPE_SELL_STOP` | Sell Stop |
| `6` | `ORDER_TYPE_BUY_STOP_LIMIT` | Buy Stop Limit |
| `7` | `ORDER_TYPE_SELL_STOP_LIMIT` | Sell Stop Limit |
| `8` | `ORDER_TYPE_CLOSE_BY` | Close by opposite |

### Order States

| Value | Constant | Description |
|---|---|---|
| `0` | `ORDER_STATE_STARTED` | Checked, not yet accepted by broker |
| `1` | `ORDER_STATE_PLACED` | Accepted |
| `2` | `ORDER_STATE_CANCELED` | Canceled by client |
| `3` | `ORDER_STATE_PARTIAL` | Partially executed |
| `4` | `ORDER_STATE_FILLED` | Fully executed |
| `5` | `ORDER_STATE_REJECTED` | Rejected |
| `6` | `ORDER_STATE_EXPIRED` | Expired |

### Order Filling Types

| Value | Constant | Description |
|---|---|---|
| `0` | `ORDER_FILLING_FOK` | Fill Or Kill |
| `1` | `ORDER_FILLING_IOC` | Immediately Or Cancel |
| `2` | `ORDER_FILLING_RETURN` | Return remaining volume |
| `3` | `ORDER_FILLING_BOC` | Book Or Cancel |

### Order Time Types

| Value | Constant | Description |
|---|---|---|
| `0` | `ORDER_TIME_GTC` | Good Till Cancel |
| `1` | `ORDER_TIME_DAY` | Good Till End of Day |
| `2` | `ORDER_TIME_SPECIFIED` | Good Till Specified Time |
| `3` | `ORDER_TIME_SPECIFIED_DAY` | Good Till Specified Day |

### Deal Types

| Value | Constant | Description |
|---|---|---|
| `0` | `DEAL_TYPE_BUY` | Buy |
| `1` | `DEAL_TYPE_SELL` | Sell |
| `2` | `DEAL_TYPE_BALANCE` | Balance operation |
| `3` | `DEAL_TYPE_CREDIT` | Credit operation |
| `4`-`17` | Various | Charge, correction, bonus, commissions, dividends, tax |

### Deal Entry Types

| Value | Constant | Description |
|---|---|---|
| `0` | `DEAL_ENTRY_IN` | Entry in (open position) |
| `1` | `DEAL_ENTRY_OUT` | Entry out (close position) |
| `2` | `DEAL_ENTRY_INOUT` | Reverse |
| `3` | `DEAL_ENTRY_OUT_BY` | Close by opposite position |

### Deal Reasons

| Value | Constant | Description |
|---|---|---|
| `0` | `DEAL_REASON_CLIENT` | Desktop terminal |
| `1` | `DEAL_REASON_MOBILE` | Mobile app |
| `2` | `DEAL_REASON_WEB` | Web platform |
| `3` | `DEAL_REASON_EXPERT` | Expert Advisor / script |
| `4` | `DEAL_REASON_SL` | Stop Loss |
| `5` | `DEAL_REASON_TP` | Take Profit |
| `6` | `DEAL_REASON_SO` | Stop Out |
| `7` | `DEAL_REASON_ROLLOVER` | Rollover |
| `8` | `DEAL_REASON_VMARGIN` | Variation margin |
| `9` | `DEAL_REASON_SPLIT` | Split |

### Tick Flags

| Value | Constant | Description |
|---|---|---|
| `0x02` | `TICK_FLAG_BID` | Bid changed |
| `0x04` | `TICK_FLAG_ASK` | Ask changed |
| `0x08` | `TICK_FLAG_LAST` | Last price changed |
| `0x10` | `TICK_FLAG_VOLUME` | Volume changed |
| `0x20` | `TICK_FLAG_BUY` | Buy tick |
| `0x40` | `TICK_FLAG_SELL` | Sell tick |

### Symbol Trade Modes

| Value | Constant | Description |
|---|---|---|
| `0` | `SYMBOL_TRADE_MODE_DISABLED` | Disabled |
| `1` | `SYMBOL_TRADE_MODE_LONGONLY` | Long only |
| `2` | `SYMBOL_TRADE_MODE_SHORTONLY` | Short only |
| `3` | `SYMBOL_TRADE_MODE_CLOSEONLY` | Close only |
| `4` | `SYMBOL_TRADE_MODE_FULL` | Full access |

### Account Trade Modes

| Value | Constant | Description |
|---|---|---|
| `0` | `ACCOUNT_TRADE_MODE_DEMO` | Demo account |
| `1` | `ACCOUNT_TRADE_MODE_CONTEST` | Contest account |
| `2` | `ACCOUNT_TRADE_MODE_REAL` | Real account |

---

## PyEventBT Internal Mapping Summary

| MT5 API call | MT5 return type | PyEventBT entity | Source file |
|---|---|---|---|
| `mt5.order_send()` | named tuple | `OrderSendResult` | `broker/mt5_broker/core/entities/order_send_result.py` |
| `result.request` | named tuple | `TradeRequest` | `broker/mt5_broker/core/entities/trade_request.py` |
| `mt5.history_deals_get()` | tuple of named tuples | `TradeDeal` | `broker/mt5_broker/core/entities/trade_deal.py` |
| `mt5.positions_get()` | tuple of named tuples | `TradePosition` | `broker/mt5_broker/core/entities/trade_position.py` |
| `mt5.orders_get()` | tuple of named tuples | `TradeOrder` | `broker/mt5_broker/core/entities/trade_order.py` |
| `mt5.account_info()` | named tuple | `AccountInfo` | `broker/mt5_broker/core/entities/account_info.py` |
| `mt5.terminal_info()` | named tuple | `TerminalInfo` | `broker/mt5_broker/core/entities/terminal_info.py` |
| `mt5.symbol_info()` | named tuple | `SymbolInfo` | `broker/mt5_broker/core/entities/symbol_info.py` |
| `mt5.symbol_info_tick()` | named tuple | `Tick` | `broker/mt5_broker/core/entities/tick.py` |
| `mt5.copy_rates_from_pos()` | numpy array | `BarEvent` / `Bar` | `events/events.py` |
| `mt5.last_error()` | tuple(int, str) | raw tuple | — |

### Conversion Pattern (live connector)

Real MT5 named tuples are converted to PyEventBT Pydantic models:

```python
# Real MT5 returns a named tuple
result = mt5.order_send(request)

# Convert to dict, then to Pydantic model
dict_result = result._asdict()
dict_result['request'] = result.request._asdict()
order_send_result = OrderSendResult(**dict_result)
```

### Simulator vs Live Differences

| Aspect | Simulator | Live |
|---|---|---|
| `retcode` | Always `10009` (DONE) | Any valid retcode |
| `deal` field | Incremented counter | May be `0` (known bug) |
| `order` field | Incremented counter | Server-assigned ticket |
| Partial fills | Not simulated | Possible |
| Requotes | Not simulated | Possible |
| Commission | Calculated from symbol config | Charged by broker |
| Swap | Charged on rollover simulation | Charged by broker |
| Slippage | No slippage | Variable |
| `exchange` in FillEvent | `"MT5_SIM"` | `"MT5"` |
