# File: `pyeventbt/broker/mt5_broker/core/interfaces/mt5_broker_interface.py`

## Module
`pyeventbt.broker.mt5_broker.core.interfaces.mt5_broker_interface`

## Purpose
Defines the abstract interface contracts for the MT5 broker layer using Python's `typing.Protocol`. Contains 8 Protocol classes that collectively define the full MT5 API surface expected by the broker layer. Connector classes (simulator and live) implement subsets of these interfaces.

## Tags
`interface`, `protocol`, `abstract`, `mt5-api`, `contract`, `design-pattern`

## Dependencies

| Dependency | Type | Purpose |
|---|---|---|
| `typing.Protocol` | Stdlib | Protocol-based structural subtyping |

## Classes/Functions

### `class IPlatform(Protocol)`

Platform lifecycle interface.

| Method | Signature | Description |
|---|---|---|
| `initialize` | `(self)` | Establish connection to MT5 terminal |
| `login` | `(self)` | Log in to a trading account |
| `shutdown` | `(self)` | Close the MT5 terminal connection |
| `version` | `(self)` | Get terminal version information |
| `last_error` | `(self)` | Get last error code and message |

### `class IAccountInfo(Protocol)`

Account information interface.

| Method | Signature | Description |
|---|---|---|
| `account_info` | `(self)` | Retrieve account information |

### `class ITerminalInfo(Protocol)`

Terminal information interface.

| Method | Signature | Description |
|---|---|---|
| `terminal_info` | `(self)` | Retrieve terminal status and properties |

### `class ISymbol(Protocol)`

Symbol data interface.

| Method | Signature | Description |
|---|---|---|
| `symbols_total` | `(self)` | Get total number of available symbols |
| `symbols_get` | `(self)` | Get symbols matching a filter |
| `symbol_info` | `(self)` | Get detailed info for a specific symbol |
| `symbol_info_tick` | `(self)` | Get latest tick for a symbol |
| `symbol_select` | `(self)` | Select/deselect a symbol in MarketWatch |

### `class IMarketBook(Protocol)`

Market depth (Level 2) interface. **No connector implementation exists.**

| Method | Signature | Description |
|---|---|---|
| `market_book_add` | `(self)` | Subscribe to Depth of Market for a symbol |
| `market_book_get` | `(self)` | Get current Depth of Market data |
| `market_book_release` | `(self)` | Unsubscribe from Depth of Market |

### `class IMarketData(Protocol)`

Historical market data interface. **No simulator connector implementation exists.**

| Method | Signature | Description |
|---|---|---|
| `copy_rates_from` | `(self)` | Copy bars from a specified date |
| `copy_rates_from_pos` | `(self)` | Copy bars from a specified position |
| `copy_rates_range` | `(self)` | Copy bars within a date range |
| `copy_ticks_from` | `(self)` | Copy ticks from a specified date |
| `copy_ticks_range` | `(self)` | Copy ticks within a date range |

### `class IOrder(Protocol)`

Order management interface. **No simulator connector implementation exists.**

| Method | Signature | Description |
|---|---|---|
| `orders_total` | `(self)` | Get total number of active orders |
| `orders_get` | `(self)` | Get active orders (with optional filters) |
| `order_calc_margin` | `(self)` | Calculate margin required for an order |
| `order_calc_profit` | `(self)` | Calculate potential profit for an order |
| `order_check` | `(self)` | Check order validity without placing |
| `order_send` | `(self)` | Send a trade order for execution |

### `class IPosition(Protocol)`

Position query interface. **No simulator connector implementation exists.**

| Method | Signature | Description |
|---|---|---|
| `positions_total` | `(self)` | Get total number of open positions |
| `positions_get` | `(self)` | Get open positions (with optional filters) |

### `class IHistory(Protocol)`

Trade history interface. **No simulator connector implementation exists.**

| Method | Signature | Description |
|---|---|---|
| `history_orders_total` | `(self)` | Get total number of historical orders |
| `history_orders_get` | `(self)` | Get historical orders |
| `history_deals_total` | `(self)` | Get total number of historical deals |
| `history_deals_get` | `(self)` | Get historical deals |

## Data Flow

```
Interface definitions (Protocol classes)
    |
    +--> PlatformConnector(IPlatform)          [Implemented in mt5_simulator_connector.py]
    +--> AccountInfoConnector(IAccountInfo)     [Implemented in mt5_simulator_connector.py]
    +--> TerminalInfoConnector(ITerminalInfo)   [Implemented in mt5_simulator_connector.py]
    +--> SymbolConnector(ISymbol)               [Implemented in mt5_simulator_connector.py]
    +--> IMarketBook                            [NOT IMPLEMENTED - raises NotImplementedError in wrapper]
    +--> IMarketData                            [NOT IMPLEMENTED]
    +--> IOrder                                 [NOT IMPLEMENTED in simulator connector]
    +--> IPosition                              [NOT IMPLEMENTED in simulator connector]
    +--> IHistory                               [NOT IMPLEMENTED in simulator connector]
```

## Gaps & Issues

1. **No parameter signatures** -- All Protocol methods accept only `self` with no typed parameters, return types, or docstrings. This makes the interfaces weak contracts that do not enforce method signatures on implementations.
2. **`raise NotImplementedError()` in Protocol methods** -- Protocol methods should typically use `...` (Ellipsis) as the body. Using `raise NotImplementedError()` is a runtime behavior that does not add value for structural subtyping.
3. **5 of 8 interfaces unimplemented** -- `IMarketBook`, `IMarketData`, `IOrder`, `IPosition`, and `IHistory` have no corresponding connector classes in `mt5_simulator_connector.py`. The source file contains a comment acknowledging this: "There are unimplemented classes in mt5_broker_interface.py for the current simulator connector."
4. **No generic base** -- There is no single `IBroker` interface composing all sub-interfaces, making it difficult to type-check a complete broker implementation.
5. **Live broker does not use these interfaces** -- `LiveMT5Broker` does not inherit from any of these Protocol classes; it calls the real `mt5` module directly.

## Requirements Derived

- **REQ-IFACE-001**: All Protocol classes should define complete method signatures (parameters + return types) to serve as meaningful contracts.
- **REQ-IFACE-002**: Simulator connector implementations should exist for all interfaces needed during backtesting (at minimum: `IOrder.order_send`, `IPosition.positions_get`).
- **REQ-IFACE-003**: The interface module should remain the single source of truth for the MT5 API surface expected by the framework.
