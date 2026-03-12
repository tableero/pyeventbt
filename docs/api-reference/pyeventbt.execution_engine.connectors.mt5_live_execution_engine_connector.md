# pyeventbt.execution_engine.connectors.mt5_live_execution_engine_connector

## File
`pyeventbt/execution_engine/connectors/mt5_live_execution_engine_connector.py`

## Module
`pyeventbt.execution_engine.connectors.mt5_live_execution_engine_connector`

## Purpose
Implements the live MT5 broker connector. Sends real orders to the MetaTrader 5 terminal, handles deal confirmation with retry logic, supports market orders, pending orders, partial closes, SL/TP updates, and futures continuous trading (CONT order type). Emits `FillEvent`s back onto the event queue after successful execution.

## Tags
`live-trading`, `mt5`, `broker-connector`, `execution`, `order-management`

## Dependencies

| Dependency | Usage |
|---|---|
| `MetaTrader5` | External MT5 Python package (conditionally imported; `mt5 = None` if unavailable) |
| `pyeventbt.execution_engine.core.interfaces.execution_engine_interface.IExecutionEngine` | Interface implemented |
| `pyeventbt.execution_engine.core.configurations.execution_engine_configurations.MT5LiveExecutionConfig` | Config for initialization |
| `pyeventbt.data_provider.core.interfaces.data_provider_interface.IDataProvider` | Bar data access |
| `pyeventbt.broker.mt5_broker.core.entities.order_send_result.OrderSendResult` | Result wrapper |
| `pyeventbt.broker.mt5_broker.core.entities.trade_deal.TradeDeal` | Deal entity for fill generation |
| `pyeventbt.events.events` | `BarEvent`, `FillEvent`, `OrderEvent` |
| `pyeventbt.portfolio.core.entities.open_position.OpenPosition` | Domain entity |
| `pyeventbt.portfolio.core.entities.pending_order.PendingOrder` | Domain entity |
| `pyeventbt.utils.utils` | `Utils`, `check_platform_compatibility` |
| `queue.Queue` | Event queue |
| `decimal.Decimal` | Precise volume/price handling |
| `time` | Retry delay (`time.sleep`) |

## Classes/Functions

### `Mt5LiveExecutionEngineConnector`

| Field | Value |
|---|---|
| **Signature** | `class Mt5LiveExecutionEngineConnector(IExecutionEngine)` |
| **Description** | Live MT5 broker connector. Sends orders via `mt5.order_send()`, polls for deal confirmation, and converts MT5 native objects to domain entities. |

#### Key Attributes

| Attribute | Type | Description |
|---|---|---|
| `events_queue` | `Queue` | Shared event queue |
| `DATA_PROVIDER` | `IDataProvider` | Data provider |
| `pending_orders` | `list[OrderSendResult]` | Client-side list of pending order results |
| `magic_number` | `int` | Strategy identifier for filtering positions/orders |

#### Core Methods

| Method | Signature | Description |
|---|---|---|
| `__init__` | `(configs, events_queue, data_provider)` | Stores config, queue, data provider; initializes pending orders list |
| `_check_common_trade_values` | `(volume, price, stop_loss, take_profit, magic, deviation, comment) -> bool` | Validates trade parameters (volume > 0, prices >= 0, comment <= 31 chars) |
| `_check_succesful_order_execution` | `(result: OrderSendResult) -> bool` | Checks if result retcode is `TRADE_RETCODE_DONE` or `TRADE_RETCODE_NO_CHANGES` |
| `_generate_and_put_fill_event` | `(trade_deal: TradeDeal, events_queue: Queue) -> None` | Creates `FillEvent` from a `TradeDeal` and enqueues it |
| `_check_if_pending_orders_filled` | `(bar_event: BarEvent) -> None` | Checks if pending orders became positions; retries deal fetch up to 1s (20 x 50ms) |
| `_get_desired_trade_method` | `(order_event: OrderEvent) -> Callable` | Returns appropriate trade method based on `buffer_data` presence |
| `get_strategy_open_volume_by_symbol` | `(symbol: str) -> Decimal` | Sums net open volume (long - short) for a symbol |
| `_process_order_event` | `(order_event: OrderEvent) -> None` | Routes to market, pending, or continuous handler based on `order_type` |
| `_update_values_and_check_executions_and_fills` | `(bar_event: BarEvent) -> None` | **No-op** (`pass`) |
| `_send_market_order` | `(order_event: OrderEvent) -> OrderSendResult` | Sends `TRADE_ACTION_DEAL`, retries deal fetch up to 5s (100 x 50ms), emits `FillEvent` |
| `_send_pending_order` | `(order_event: OrderEvent) -> OrderSendResult` | Sends `TRADE_ACTION_PENDING` (LIMIT/STOP), stores in `pending_orders` |
| `close_position` | `(position_ticket: int, partial_volume: Decimal = 0.0) -> OrderSendResult` | Closes position (full or partial), retries deal fetch, emits `FillEvent` |
| `close_all_strategy_positions` | `() -> None` | Iterates all positions, closes those matching `magic_number` |
| `close_all_strategy_positions_by_symbol` | `(symbol: str) -> None` | Closes strategy positions for a specific symbol |
| `close_strategy_long_positions_by_symbol` | `(symbol: str) -> None` | Closes BUY positions matching strategy + symbol |
| `close_strategy_short_positions_by_symbol` | `(symbol: str) -> None` | Closes SELL positions matching strategy + symbol |
| `cancel_pending_order` | `(order_ticket: int) -> OrderSendResult` | Sends `TRADE_ACTION_REMOVE` to cancel a pending order |
| `cancel_all_strategy_pending_orders` | `() -> None` | Cancels all pending orders matching `magic_number` |
| `cancel_all_strategy_pending_orders_by_type_and_symbol` | `(order_type: str, symbol: str) -> None` | Cancels filtered pending orders using `Utils.order_type_str_to_int` |
| `update_position_sl_tp` | `(position_ticket, new_sl, new_tp) -> None` | Sends `TRADE_ACTION_SLTP` to modify SL/TP |
| `_get_account_currency` | `() -> str` | Returns currency from `mt5.account_info()` |
| `_get_account_balance` | `() -> Decimal` | Returns balance from `mt5.account_info()` |
| `_get_account_equity` | `() -> Decimal` | Returns equity from `mt5.account_info()` |
| `_get_account_floating_profit` | `() -> Decimal` | Returns profit from `mt5.account_info()` |
| `_get_account_used_margin` | `() -> Decimal` | Returns margin from `mt5.account_info()` |
| `_get_account_free_margin` | `() -> Decimal` | Returns margin_free from `mt5.account_info()` |
| `_get_strategy_pending_orders` | `(symbol, ticket, group) -> tuple[PendingOrder]` | Fetches orders via `mt5.orders_get()`, filters by magic, converts to `PendingOrder` |
| `_get_strategy_positions` | `(symbol, ticket, group) -> tuple[OpenPosition]` | Fetches positions via `mt5.positions_get()`, filters by magic, converts to `OpenPosition` |
| `_get_symbol_min_volume` | `(symbol: str) -> Decimal` | Returns `mt5.symbol_info(symbol).volume_min` |

## Data Flow

```
OrderEvent --> _process_order_event()
    |
    +--> MARKET: _send_market_order()
    |       --> mt5.order_send(TRADE_ACTION_DEAL)
    |       --> retry loop: mt5.history_deals_get(position=result.order) up to 5s
    |       --> for each deal: _generate_and_put_fill_event() --> FillEvent --> queue
    |
    +--> LIMIT/STOP: _send_pending_order()
    |       --> mt5.order_send(TRADE_ACTION_PENDING)
    |       --> store OrderSendResult in self.pending_orders
    |
    +--> CONT: execute_desired_continuous_trade()

BarEvent --> _update_values_and_check_executions_and_fills()
    --> pass (no-op)
```

## Gaps & Issues

1. **`_update_values_and_check_executions_and_fills` is `pass`.** No proactive checking of pending order fills or SL/TP hits from the client side. Relies entirely on MT5 server-side processing.
2. **Deal confirmation retry is blocking.** Up to 5 seconds of `time.sleep(0.05)` in a loop blocks the event loop. No async support.
3. **`_check_if_pending_orders_filled` not called from `_update_values_and_check_executions_and_fills`.** The method exists but is never invoked in the standard bar-processing flow.
4. **MT5 `result.deal` returns 0 workaround.** Comments note a MetaQuotes bug where live accounts return 0 in the deal field; code uses `position=result.order` as a workaround.
5. **`close_position` returns `False` (bool) on error** instead of an `OrderSendResult`, breaking the return type contract.
6. **Duplicate `_check_common_trade_values`.** Same validation logic exists in the simulator connector.
7. **Magic number cast.** `int(self.magic_number)` is called repeatedly even though `magic_number` is already `int` from the config.
8. **Typo.** Method `_check_succesful_order_execution` -- "succesful" should be "successful".

## Requirements Derived

- R-EXEC-LIVE-01: Market orders shall be sent via `mt5.order_send` with `TRADE_ACTION_DEAL` and `ORDER_FILLING_FOK`.
- R-EXEC-LIVE-02: Deal confirmation shall be retried for up to 5 seconds before giving up.
- R-EXEC-LIVE-03: Pending orders shall be sent via `TRADE_ACTION_PENDING` and tracked client-side.
- R-EXEC-LIVE-04: Position and order queries shall filter by strategy `magic_number`.
- R-EXEC-LIVE-05: The connector shall handle missing MT5 package gracefully (conditional import, `mt5 = None`).
- R-EXEC-LIVE-06: `FillEvent`s shall be generated from `TradeDeal` objects, mapping `entry == 0` to "IN" and otherwise to "OUT".
