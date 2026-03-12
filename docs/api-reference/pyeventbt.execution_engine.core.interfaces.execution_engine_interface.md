# pyeventbt.execution_engine.core.interfaces.execution_engine_interface

## File
`pyeventbt/execution_engine/core/interfaces/execution_engine_interface.py`

## Module
`pyeventbt.execution_engine.core.interfaces.execution_engine_interface`

## Purpose
Defines the `IExecutionEngine` interface that all execution engine connectors must implement. Covers order processing, position/order management, and account queries. This is the contract shared by both the simulated and live MT5 connectors.

## Tags
`interface`, `execution`, `abstract`, `order-management`, `account-queries`

## Dependencies

| Dependency | Usage |
|---|---|
| `typing.Protocol` | Imported but not used as base class (class uses plain inheritance) |
| `pyeventbt.events.events.BarEvent` | Input for value update methods |
| `pyeventbt.events.events.OrderEvent` | Input for order processing methods |
| `pyeventbt.broker.mt5_broker.core.entities.order_send_result.OrderSendResult` | Return type for order operations |
| `pyeventbt.portfolio.core.entities.open_position.OpenPosition` | Return type for position queries |
| `decimal.Decimal` | Return type for monetary/volume queries |

## Classes/Functions

### `IExecutionEngine`

| Field | Value |
|---|---|
| **Signature** | `class IExecutionEngine` |
| **Description** | Interface for execution engines. Defines ~20 methods covering order execution, position management, pending order management, and account state queries. Methods raise `NotImplementedError` if not overridden. |

#### Order Processing Methods

| Method | Signature | Description | Returns |
|---|---|---|---|
| `_process_order_event` | `(self, order_event: OrderEvent) -> None` | Main entry: routes order to market, pending, or continuous handler | `None` |
| `_update_values_and_check_executions_and_fills` | `(self, bar_event: BarEvent) -> None` | Updates account values, checks pending fills and SL/TP hits on each bar | `None` |
| `_send_market_order` | `(self, order_event: OrderEvent) -> OrderSendResult` | Executes a market order immediately | `OrderSendResult` |
| `_send_pending_order` | `(self, order_event: OrderEvent) -> OrderSendResult` | Places a limit/stop order | `OrderSendResult` |

#### Position Management Methods

| Method | Signature | Description | Returns |
|---|---|---|---|
| `close_position` | `(self, position_ticket: int) -> OrderSendResult` | Closes a single position by ticket | `OrderSendResult` |
| `close_all_strategy_positions` | `(self) -> None` | Closes all positions for the strategy | `None` |
| `close_strategy_long_positions_by_symbol` | `(self, symbol: str) -> None` | Closes all long positions for a symbol | `None` |
| `close_strategy_short_positions_by_symbol` | `(self, symbol: str) -> None` | Closes all short positions for a symbol | `None` |
| `update_position_sl_tp` | `(self, position_ticket: int, new_sl: float, new_tp: float) -> None` | Updates SL/TP on an open position | `None` |

#### Pending Order Management Methods

| Method | Signature | Description | Returns |
|---|---|---|---|
| `cancel_pending_order` | `(self, order_ticket: int) -> OrderSendResult` | Cancels a single pending order | `OrderSendResult` |
| `cancel_all_strategy_pending_orders` | `(self) -> None` | Cancels all strategy pending orders | `None` |
| `cancel_all_strategy_pending_orders_by_type_and_symbol` | `(self, order_type: str, symbol: str) -> None` | Cancels pending orders filtered by type and symbol | `None` |

#### Account Query Methods

| Method | Signature | Description | Returns |
|---|---|---|---|
| `_get_account_currency` | `(self) -> str` | Account denomination currency | `str` |
| `_get_account_balance` | `(self) -> Decimal` | Current account balance | `Decimal` |
| `_get_account_equity` | `(self) -> Decimal` | Current account equity | `Decimal` |
| `_get_account_floating_profit` | `(self) -> Decimal` | Unrealized P&L | `Decimal` |
| `_get_account_used_margin` | `(self) -> Decimal` | Margin in use | `Decimal` |
| `_get_account_free_margin` | `(self) -> Decimal` | Available margin | `Decimal` |
| `_get_total_number_of_pending_orders` | `(self) -> int` | Count of active pending orders | `int` |
| `_get_strategy_pending_orders` | `(self) -> tuple` | Tuple of strategy's pending orders | `tuple` |
| `_get_total_number_of_positions` | `(self) -> int` | Count of open positions | `int` |
| `_get_strategy_positions` | `(self) -> tuple[OpenPosition]` | Tuple of strategy's open positions | `tuple[OpenPosition]` |
| `_get_symbol_min_volume` | `(self, symbol: str) -> Decimal` | Minimum trade volume for a symbol | `Decimal` |

#### Toggle Methods

| Method | Signature | Description | Returns |
|---|---|---|---|
| `enable_trading` | `(self) -> None` | Enables order processing (no-op in interface) | `None` |
| `disable_trading` | `(self) -> None` | Disables order processing (no-op in interface) | `None` |

## Data Flow

```
TradingDirector dispatches OrderEvent
    --> IExecutionEngine._process_order_event()
        --> _send_market_order() or _send_pending_order()
            --> FillEvent emitted to queue (on success)

TradingDirector dispatches BarEvent
    --> IExecutionEngine._update_values_and_check_executions_and_fills()
        --> checks pending order fills, SL/TP hits
        --> may emit FillEvents
```

## Gaps & Issues

1. **Not a true ABC or Protocol.** The class does not inherit from `abc.ABC` or use `@abstractmethod`, nor does it properly use `typing.Protocol`. It relies on `raise NotImplementedError()` convention.
2. **Underscore-prefixed public methods.** Many methods used by `PortfolioHandler` and other external components are prefixed with `_` (e.g., `_get_account_balance`), suggesting they are private, but they are part of the cross-component API.
3. **Commented-out method.** `check_if_pending_orders_filled` is commented out but its logic exists in connectors under different names.
4. **Docstring typo.** `_get_total_number_of_positions` has the docstring "Get total number of active pending orders" instead of "positions."
5. **`enable_trading`/`disable_trading` are `pass`.** They do nothing in the interface; the actual toggle logic lives only in `ExecutionEngine` (service layer).

## Requirements Derived

- R-EXEC-IF-01: All execution connectors shall implement the full `IExecutionEngine` method set.
- R-EXEC-IF-02: Order processing methods shall emit `FillEvent`s on successful execution.
- R-EXEC-IF-03: Account query methods shall return current values reflecting all executed trades.
- R-EXEC-IF-04: Position and pending order management methods shall filter by strategy magic number.
