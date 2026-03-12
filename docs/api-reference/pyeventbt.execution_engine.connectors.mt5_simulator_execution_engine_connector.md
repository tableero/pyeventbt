# pyeventbt.execution_engine.connectors.mt5_simulator_execution_engine_connector

## File
`pyeventbt/execution_engine/connectors/mt5_simulator_execution_engine_connector.py`

## Module
`pyeventbt.execution_engine.connectors.mt5_simulator_execution_engine_connector`

## Purpose
Implements a full MT5-compatible backtest simulator. Manages in-memory account state, positions, pending orders, and deals. Handles market and pending order execution, SL/TP checking, margin tracking, swap/commission calculation, and multi-currency P&L conversion. This is the primary connector for backtesting strategies.

## Tags
`simulator`, `backtesting`, `mt5`, `execution`, `p&l`, `margin`, `swap`, `commission`

## Dependencies

| Dependency | Usage |
|---|---|
| `pyeventbt.execution_engine.core.interfaces.execution_engine_interface.IExecutionEngine` | Interface implemented |
| `pyeventbt.execution_engine.core.configurations.execution_engine_configurations.MT5SimulatedExecutionConfig` | Config for initialization |
| `pyeventbt.data_provider.core.interfaces.data_provider_interface.IDataProvider` | Bar data for price lookups |
| `pyeventbt.broker.mt5_broker.core.entities.*` | `OrderSendResult`, `TradePosition`, `TradeRequest`, `TradeOrder`, `TradeDeal`, `AccountInfo` |
| `pyeventbt.broker.mt5_broker.mt5_simulator_wrapper.Mt5SimulatorWrapper` | Imported as `mt5`, provides simulated MT5 API |
| `pyeventbt.broker.mt5_broker.shared.shared_data.SharedData` | Shared state for account info |
| `pyeventbt.portfolio.core.entities.open_position.OpenPosition` | Domain entity for position queries |
| `pyeventbt.portfolio.core.entities.pending_order.PendingOrder` | Domain entity for pending order queries |
| `pyeventbt.events.events` | `BarEvent`, `FillEvent`, `OrderEvent`, `SignalType` |
| `pyeventbt.utils.utils.Utils` | Utility functions |
| `queue.Queue` | Event queue |
| `decimal.Decimal` | Precise arithmetic |
| `datetime` | Timestamp handling |

## Classes/Functions

### `Mt5SimulatorExecutionEngineConnector`

| Field | Value |
|---|---|
| **Signature** | `class Mt5SimulatorExecutionEngineConnector(IExecutionEngine)` |
| **Description** | Full MT5 backtesting simulator. Manages positions, orders, and deals in-memory. Calculates P&L, swap, commission, and margin. Emits `FillEvent`s on trade execution. |

#### Key Attributes

| Attribute | Type | Description |
|---|---|---|
| `events_queue` | `Queue` | Shared event queue |
| `DATA_PROVIDER` | `IDataProvider` | Data provider for price data |
| `pending_orders` | `dict[int, OrderSendResult]` | Active pending orders keyed by ticket |
| `open_positions` | `dict[int, TradePosition]` | Open positions keyed by ticket |
| `executed_deals` | `dict[int, TradeDeal]` | Historical deals keyed by ticket |
| `balance` | `Decimal` | Account balance |
| `equity` | `Decimal` | Account equity (balance + floating P&L) |
| `used_margin` | `Decimal` | Margin currently in use |
| `free_margin` | `Decimal` | Available margin |
| `account_currency` | `str` | Account denomination currency |
| `ticketing_counter` | `int` | Auto-incrementing ticket for positions (starts 200000000) |
| `deal_ticketing_counter` | `int` | Auto-incrementing ticket for deals (starts 300000000) |
| `margin_call` | `bool` | Margin call flag |
| `all_fx_symbols` | `tuple` | Known FX pair symbols (30 pairs) |
| `all_commodities_symbols` | `tuple` | Known commodity symbols (4) |
| `all_indices_symbols` | `tuple` | Known index symbols (10) |

#### Core Methods

| Method | Description |
|---|---|
| `__init__(configs, events_queue, data_provider)` | Initializes account state, data structures, and shared data |
| `_check_common_trade_values(...)` | Validates volume, price, SL, TP, magic, deviation, comment |
| `_process_order_event(order_event)` | Routes to market, pending, or continuous order handler |
| `_send_market_order(order_event)` | Executes market order, creates position and deal, emits `FillEvent` |
| `_send_pending_order(order_event)` | Creates pending order entry in `pending_orders` dict |
| `_update_values_and_check_executions_and_fills(bar_event)` | Per-bar: updates floating P&L, checks pending triggers, checks SL/TP |
| `close_position(position_ticket)` | Closes a position, calculates final P&L, emits `FillEvent` |
| `close_all_strategy_positions()` | Closes all open positions for the strategy |
| `close_strategy_long_positions_by_symbol(symbol)` | Closes long positions filtered by symbol |
| `close_strategy_short_positions_by_symbol(symbol)` | Closes short positions filtered by symbol |
| `cancel_pending_order(order_ticket)` | Removes a pending order |
| `cancel_all_strategy_pending_orders()` | Removes all strategy pending orders |
| `cancel_all_strategy_pending_orders_by_type_and_symbol(order_type, symbol)` | Removes filtered pending orders |
| `update_position_sl_tp(position_ticket, new_sl, new_tp)` | Modifies SL/TP on an open position |
| `_get_account_balance()` | Returns `self.balance` |
| `_get_account_equity()` | Returns `self.equity` |
| `_get_account_currency()` | Returns `self.account_currency` |
| `_get_account_floating_profit()` | Returns equity minus balance |
| `_get_account_used_margin()` | Returns `self.used_margin` |
| `_get_account_free_margin()` | Returns `self.free_margin` |
| `_get_strategy_positions()` | Returns tuple of `OpenPosition` domain entities |
| `_get_strategy_pending_orders()` | Returns tuple of `PendingOrder` domain entities |
| `_get_symbol_min_volume(symbol)` | Returns minimum volume via `mt5.symbol_info` |
| `_update_shared_data_account_info()` | Syncs account state to `SharedData` |

## Data Flow

```
OrderEvent --> _process_order_event()
    |
    +--> MARKET: _send_market_order()
    |       --> create TradePosition in open_positions
    |       --> create TradeDeal in executed_deals
    |       --> update balance/equity/margin
    |       --> FillEvent --> events_queue
    |
    +--> LIMIT/STOP: _send_pending_order()
            --> store in pending_orders dict

BarEvent --> _update_values_and_check_executions_and_fills()
    |
    +--> update floating P&L for all open positions
    +--> check pending orders against current bar prices
    |       --> if triggered: convert to position, emit FillEvent
    +--> check SL/TP for all open positions
            --> if hit: close position, emit FillEvent
```

## Gaps & Issues

1. **Margin check TODO.** Line 37: "TODO: Add Check if margin is enough to keep positions opened." Margin is tracked but never enforced for new orders or existing positions.
2. **No automated tests.** Complex P&L calculation (including cross-currency conversion), swap, commission, and pending order trigger logic has zero test coverage.
3. **Hardcoded symbol lists.** FX pairs, commodities, and indices are hardcoded tuples. Adding new instruments requires code changes.
4. **Duplicated validation.** `_check_common_trade_values` is copy-pasted between simulator and live connectors.
5. **Large file size.** At 1000+ lines, the module handles too many responsibilities (account management, order execution, P&L calculation, pending order checking, SL/TP monitoring).

## Requirements Derived

- R-EXEC-SIM-01: The simulator shall maintain in-memory account state (balance, equity, margin) reflecting all executed trades.
- R-EXEC-SIM-02: Market orders shall be filled at current bar prices and immediately emit `FillEvent`s.
- R-EXEC-SIM-03: Pending orders shall be checked against each new bar and triggered when price conditions are met.
- R-EXEC-SIM-04: SL/TP levels shall be checked on each bar and positions closed automatically when hit.
- R-EXEC-SIM-05: P&L shall be calculated with proper currency conversion when the instrument's profit currency differs from the account currency.
- R-EXEC-SIM-06: Swap and commission shall be applied to positions according to instrument specifications.
