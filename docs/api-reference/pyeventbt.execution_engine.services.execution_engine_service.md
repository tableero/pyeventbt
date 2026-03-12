# pyeventbt.execution_engine.services.execution_engine_service

## File
`pyeventbt/execution_engine/services/execution_engine_service.py`

## Module
`pyeventbt.execution_engine.services.execution_engine_service`

## Purpose
Provides the `ExecutionEngine` facade that serves as the single entry point for all execution operations. Selects the appropriate connector (simulator or live) based on config type, delegates all `IExecutionEngine` methods 1:1, and adds an `enable_trading`/`disable_trading` toggle that gates order processing.

## Tags
`service`, `facade`, `execution`, `delegation`, `trading-toggle`

## Dependencies

| Dependency | Usage |
|---|---|
| `pyeventbt.execution_engine.core.interfaces.execution_engine_interface.IExecutionEngine` | Interface implemented and delegated to |
| `pyeventbt.execution_engine.core.configurations.execution_engine_configurations` | `BaseExecutionConfig`, `MT5SimulatedExecutionConfig`, `MT5LiveExecutionConfig` for factory logic |
| `pyeventbt.execution_engine.connectors.mt5_live_execution_engine_connector.Mt5LiveExecutionEngineConnector` | Live connector |
| `pyeventbt.execution_engine.connectors.mt5_simulator_execution_engine_connector.Mt5SimulatorExecutionEngineConnector` | Simulator connector |
| `pyeventbt.broker.mt5_broker.core.entities.order_send_result.OrderSendResult` | Return type |
| `pyeventbt.data_provider.core.interfaces.data_provider_interface.IDataProvider` | Passed to connectors |
| `pyeventbt.events.events` | `BarEvent`, `OrderEvent`, `FillEvent` |
| `pyeventbt.utils.utils` | `TerminalColors`, `colorize` |
| `queue.Queue` | Shared event queue |
| `logging` | Logger for warning messages |

## Classes/Functions

### `ExecutionEngine`

| Field | Value |
|---|---|
| **Signature** | `class ExecutionEngine(IExecutionEngine)` |
| **Description** | Facade over the execution connectors. Instantiates the correct connector based on config type and delegates all method calls. Adds a trading toggle that prevents order processing when disabled. |

#### Key Attributes

| Attribute | Type | Description |
|---|---|---|
| `events_queue` | `Queue` | Shared event queue |
| `DATA_PROVIDER` | `IDataProvider` | Data provider instance |
| `EXECUTION_ENGINE` | `IExecutionEngine` | The selected connector (simulator or live) |
| `__enable_trading` | `bool` | Private flag controlling whether orders are processed (default `True`) |

#### Methods

| Method | Signature | Description |
|---|---|---|
| `__init__` | `(events_queue: Queue, data_provider: IDataProvider, execution_config: BaseExecutionConfig) -> None` | Stores references, calls factory to create connector |
| `_get_execution_engine` | `(execution_config: BaseExecutionConfig) -> IExecutionEngine` | Factory: `MT5LiveExecutionConfig` --> `Mt5LiveExecutionEngineConnector`; `MT5SimulatedExecutionConfig` --> `Mt5SimulatorExecutionEngineConnector`; else raises `Exception` |
| `_put_fill_event` | `(fill_event: FillEvent) -> None` | Puts a fill event on the queue (utility, not in interface) |
| `enable_trading` | `() -> None` | Sets `__enable_trading = True` |
| `disable_trading` | `() -> None` | Sets `__enable_trading = False` |
| `_process_order_event` | `(order_event: OrderEvent) -> None` | Delegates to connector only if `__enable_trading` is `True`; logs warning otherwise |
| `_update_values_and_check_executions_and_fills` | `(bar_event: BarEvent) -> None` | Delegates to connector |
| `_send_market_order` | `(order_event: OrderEvent) -> OrderSendResult` | Delegates to connector |
| `_send_pending_order` | `(order_event: OrderEvent) -> OrderSendResult` | Delegates to connector |
| `close_position` | `(position_ticket: int) -> OrderSendResult` | Delegates to connector |
| `close_all_strategy_positions` | `() -> None` | Delegates to connector |
| `close_strategy_long_positions_by_symbol` | `(symbol: str) -> None` | Delegates to connector |
| `close_strategy_short_positions_by_symbol` | `(symbol: str) -> None` | Delegates to connector |
| `cancel_pending_order` | `(order_ticket: int) -> OrderSendResult` | Delegates to connector |
| `cancel_all_strategy_pending_orders` | `() -> None` | Delegates to connector |
| `cancel_all_strategy_pending_orders_by_type_and_symbol` | `(order_type: str, symbol: str) -> None` | Delegates to connector |
| `update_position_sl_tp` | `(position_ticket: int, new_sl: float, new_tp: float) -> None` | Delegates to connector |
| `_get_account_currency` | `() -> str` | Delegates to connector |
| `_get_account_balance` | `() -> float` | Delegates to connector (note: return type annotation is `float`, connector returns `Decimal`) |
| `_get_account_equity` | `() -> float` | Delegates to connector |
| `_get_account_floating_profit` | `() -> float` | Delegates to connector |
| `_get_account_used_margin` | `() -> float` | Delegates to connector |
| `_get_account_free_margin` | `() -> float` | Delegates to connector |
| `_get_total_number_of_pending_orders` | `() -> int` | Delegates to connector |
| `_get_strategy_pending_orders` | `() -> tuple` | Delegates to connector |
| `_get_total_number_of_positions` | `() -> int` | Delegates to connector |
| `_get_strategy_positions` | `() -> tuple` | Delegates to connector |
| `_get_symbol_min_volume` | `(symbol: str) -> float` | Delegates to connector |

## Data Flow

```
TradingDirector / PortfolioHandler
    --> ExecutionEngine (facade)
        |
        +--> _process_order_event (gated by __enable_trading)
        |       --> connector._process_order_event()
        |
        +--> all other methods: direct 1:1 delegation
                --> connector.<method>()
```

## Gaps & Issues

1. **Typo in log message.** Line 64: `"Trading is disbled"` should be `"Trading is disabled"`.
2. **Return type annotations mismatch.** Account query methods are annotated as returning `float` but the connectors return `Decimal`. The facade does not convert.
3. **Trading toggle only gates `_process_order_event`.** Other order-affecting methods like `close_position`, `cancel_pending_order`, etc. are not gated. A user could still close positions or cancel orders even when trading is "disabled."
4. **`_put_fill_event` is defined but unused.** The connectors handle fill event emission themselves.
5. **`close_position` signature mismatch.** The facade takes `(position_ticket: int)` but the live connector accepts an optional `partial_volume: Decimal` parameter, which is lost through the facade.

## Requirements Derived

- R-EXEC-SVC-01: `ExecutionEngine` shall select the appropriate connector based on the config type at construction time.
- R-EXEC-SVC-02: When trading is disabled, `_process_order_event` shall log a warning and discard the order.
- R-EXEC-SVC-03: All `IExecutionEngine` methods shall be delegated to the active connector without modification.
- R-EXEC-SVC-04: Unrecognized config types shall raise an `Exception` at construction time.
