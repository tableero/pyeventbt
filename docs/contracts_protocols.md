# Component Contracts, Behaviors & Protocols

> Extracted from the main documentation. See also: [Design Pattern](design_pattern.md) | [Distributed Migration](distributed_migration.md) | [Industry Research](industry_research.md)

This section defines a formal specification for every process, module, and step in the system. The goal is to make each component independently implementable, swappable, and composable — so you can build a DataProvider for Binance, a broker for Interactive Brokers, run multiple accounts, or mix providers freely.

### 29.1 Terminology

Three terms define how components work together:

| Term | What it defines | Example |
|---|---|---|
| **Contract** | What a component **must do** — its interface (methods, inputs, outputs, guarantees). A contract is what you code against. | "A DataProvider must implement `get_latest_bars(symbol, tf, count) → DataFrame`" |
| **Behavior** | **How** a component acts at runtime — its internal logic, state transitions, side effects, and invariants. | "When a BarEvent arrives, Portfolio marks all open positions to market before anything else happens" |
| **Protocol** | The **communication rules** between components — what messages flow, in what order, and what each side expects from the other. | "DataProvider publishes BarEvent; SignalEngine consumes it and may publish SignalEvent" |

When you want to add a new DataProvider (e.g., Binance WebSocket) or a new broker (e.g., Interactive Brokers), you implement the **contract**, respect the **behavior** invariants, and follow the **protocol** for communication.

---

### 29.2 Process 1: Data Provider

#### Contract

Any data provider must implement:

```python
class IDataProvider(Protocol):
    # ── Feeding ──────────────────────────────────────────────
    def update_bars(self) -> None:
        """Produce the next batch of BarEvents and put them on the bus.
        Called by the event loop when the queue is empty."""

    # ── Queries (used by signal/sizing engines via Modules) ──
    def get_latest_bars(self, symbol: str, timeframe: str, count: int) -> pl.DataFrame:
        """Return the last `count` completed bars as a Polars DataFrame.
        Columns: datetime, open, high, low, close, tickvol, volume, spread."""

    def get_latest_bar(self, symbol: str, timeframe: str) -> dict:
        """Return the most recent completed bar as a dict."""

    def get_latest_tick(self, symbol: str) -> dict:
        """Return the latest tick with keys: bid, ask, last, time."""

    def get_latest_bid(self, symbol: str) -> Decimal:
        """Return the current best bid price."""

    def get_latest_ask(self, symbol: str) -> Decimal:
        """Return the current best ask price."""

    def get_latest_datetime(self, symbol: str, timeframe: str) -> datetime:
        """Return the timestamp of the last completed bar."""
```

#### Behavior

| Rule | Description |
|---|---|
| **Bar ordering** | Bars must be emitted in chronological order per symbol. |
| **Multi-symbol alignment** | When feeding multiple symbols, all symbols for the same timestamp must be emitted before advancing to the next timestamp. |
| **Timeframe hierarchy** | Base timeframe bars are emitted first. Higher timeframe bars (if boundary crossed) are emitted immediately after. |
| **No lookahead** | `get_latest_bars()` for non-base timeframes must return only completed bars, never the currently forming bar. |
| **Integer encoding** | Bar OHLC prices are stored as integers (`price * 10^digits`). The `digits` field must be set correctly per symbol. |
| **Gap handling** | Missing timestamps should be filled with phantom bars (OHLC = prior close, volume = 1) and flagged so they can be skipped. |
| **End-of-data signal** | When no more data is available, the provider must signal termination (e.g., set `continue_backtest = False` or raise `StopIteration`). |

#### Protocol

```
DataProvider ──publishes──▶ BarEvent ──▶ Bus

No incoming events. DataProvider is a source-only component.
Triggered by: event loop calling update_bars() when queue is empty.
```

#### How to Add a New Data Provider

To add a new source (e.g., Binance, Yahoo Finance, a database):

1. Implement `IDataProvider` with all methods above.
2. Create a configuration class extending `BaseDataConfig`:
   ```python
   class BinanceDataConfig(BaseDataConfig):
       api_key: str
       api_secret: str
       tradeable_symbol_list: list[str]
       timeframes_list: list[str]
   ```
3. Register it in the `DataProvider` service dispatcher so the correct connector is instantiated based on config type.
4. Respect the behavior rules — especially bar ordering, alignment, and no lookahead.

#### Multi-Provider Pattern

To have **multiple data providers** feeding the same system (e.g., Binance for crypto + CSV for forex):

```
BinanceProvider ──▶ BarEvent(symbol="BTCUSDT") ──▶ Bus
CSVProvider     ──▶ BarEvent(symbol="EURUSD")  ──▶ Bus

Both feed the same queue. Downstream components route by symbol.
```

The contract stays the same per provider. A `CompositeDataProvider` can wrap multiple providers and call `update_bars()` on each:

```python
class CompositeDataProvider(IDataProvider):
    def __init__(self, providers: dict[str, IDataProvider]):
        self.providers = providers  # {"BTCUSDT": binance, "EURUSD": csv}

    def update_bars(self):
        for provider in self.providers.values():
            provider.update_bars()

    def get_latest_bars(self, symbol, tf, count):
        return self.providers[symbol].get_latest_bars(symbol, tf, count)
```

---

### 29.3 Process 2: Signal Engine

#### Contract

```python
class ISignalEngine(Protocol):
    def generate_signal(
        self, bar_event: BarEvent, modules: Modules
    ) -> SignalEvent | list[SignalEvent] | None:
        """Analyze incoming bar data and optionally produce trade signals.

        Args:
            bar_event: The new bar that triggered this call.
            modules: Access to DATA_PROVIDER (read bars/ticks),
                     PORTFOLIO (read positions), EXECUTION_ENGINE (close positions).

        Returns:
            - SignalEvent: a single trade idea
            - list[SignalEvent]: multiple trade ideas (e.g., for multiple symbols)
            - None: no signal this bar
        """
```

#### Behavior

| Rule | Description |
|---|---|
| **Pure analysis by default** | Signal engines should ideally only read state (bars, positions) and return signals. Side effects (closing positions) are allowed but create coupling. |
| **Timeframe filtering** | The engine is called on every bar. It must check `bar_event.timeframe` if it only wants to act on specific timeframes. |
| **Strategy ID scoping** | Each signal engine is bound to a `strategy_id`. It should only read/act on positions belonging to that strategy. |
| **Forecast range** | `SignalEvent.forecast` should be in the range -20 to +20 (convention, not enforced). |
| **SL/TP required for RiskPctSizing** | If the downstream sizing engine is `RiskPctSizingConfig`, the signal must set `sl != 0`. |
| **Idempotent on replay** | Given the same bar sequence, the engine should produce the same signals (deterministic for backtest reproducibility). |

#### Protocol

```
Bus ──delivers──▶ BarEvent ──▶ SignalEngine
SignalEngine ──publishes──▶ SignalEvent ──▶ Bus (or nothing)
```

**Reads from (via Modules):**
- `DATA_PROVIDER.get_latest_bars()` — historical bar data for indicator calculation
- `PORTFOLIO.get_number_of_strategy_open_positions_by_symbol()` — current exposure
- `DATA_PROVIDER.get_latest_tick()` — current bid/ask

**Side effects (optional, creates coupling):**
- `EXECUTION_ENGINE.close_strategy_short_positions_by_symbol()` — close before reversing

#### SignalEvent Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `symbol` | str | yes | Instrument to trade |
| `signal_type` | SignalType | yes | BUY or SELL |
| `order_type` | OrderType | yes | MARKET, LIMIT, or STOP |
| `strategy_id` | str | yes | Numeric string mapping to MT5 magic number |
| `sl` | Decimal | recommended | Stop loss price (required if RiskPctSizing) |
| `tp` | Decimal | optional | Take profit price |
| `order_price` | Decimal | for LIMIT/STOP | Price at which to place the order |
| `forecast` | float | optional | Signal strength, -20 to +20 |
| `rollover` | tuple | optional | `(True, old_contract, new_contract)` for futures |

---

### 29.4 Process 3: Sizing Engine

#### Contract

```python
class ISizingEngine(Protocol):
    def get_suggested_order(
        self, signal_event: SignalEvent, modules: Modules
    ) -> SuggestedOrder:
        """Convert a signal into a sized order suggestion.

        Args:
            signal_event: The trade idea to size.
            modules: Access to account state (equity, balance) and market data (tick prices).

        Returns:
            SuggestedOrder with volume calculated. Volume = 0 means the signal
            cannot be sized (e.g., insufficient equity).
        """
```

#### Behavior

| Rule | Description |
|---|---|
| **Volume must respect symbol limits** | `volume_min <= volume <= volume_max`, rounded to `volume_step`. |
| **Volume = 0 means reject** | If the engine cannot size (e.g., not enough equity), return volume = 0. Downstream risk engine will discard it. |
| **Account currency conversion** | When sizing based on risk %, the engine must convert SL distance to account currency using cross-rates. |
| **No side effects** | Sizing engines must be read-only. They read state and return a number. They must not modify positions, place orders, or emit events. |

#### Protocol

```
Bus ──delivers──▶ SignalEvent ──▶ SizingEngine
SizingEngine ──returns──▶ SuggestedOrder ──▶ (passed to RiskEngine, same process)
```

**Note:** In the current architecture, SuggestedOrder is **not** a queued event — it is passed synchronously from sizing to risk within `PortfolioHandler`. In a distributed architecture, promote it to a queued `SuggestedOrderEvent`.

#### Built-in Implementations

| Config | Behavior |
|---|---|
| `MinSizingConfig` | `volume = symbol.volume_min` — always trades the smallest allowed lot |
| `FixedSizingConfig(volume=X)` | `volume = X` — always trades a fixed lot size |
| `RiskPctSizingConfig(risk_pct=R)` | `volume = (equity * R/100) / (SL_distance * tick_value)` — risks R% of equity per trade |
| Custom (`@strategy.custom_sizing_engine`) | User-defined function with full access to Modules |

#### SuggestedOrder Fields

| Field | Type | Description |
|---|---|---|
| `signal_event` | SignalEvent | The original signal (carried through for downstream use) |
| `volume` | Decimal | Computed position size |
| `buffer_data` | dict (optional) | Arbitrary data passed through to OrderEvent and execution |

---

### 29.5 Process 4: Risk Engine

#### Contract

```python
class IRiskEngine(Protocol):
    def assess_order(
        self, suggested_order: SuggestedOrder, modules: Modules
    ) -> float:
        """Validate or filter a sized order before execution.

        Args:
            suggested_order: The sized order to validate.
            modules: Access to portfolio state for risk checks.

        Returns:
            float: The approved volume.
                - > 0: Order approved (possibly with adjusted volume)
                - = 0: Order rejected
        """
```

#### Behavior

| Rule | Description |
|---|---|
| **Gatekeeper role** | The risk engine is the last checkpoint before an order enters the execution pipeline. It can reduce volume or reject entirely. |
| **Return value semantics** | `volume > 0` → emit OrderEvent with that volume. `volume == 0` → silently discard, no event emitted. |
| **No side effects** | Risk engines must not place orders, close positions, or modify state. They only return a number. |
| **Composable** | Multiple risk checks can be chained. Each receives the previous volume and can reduce it further. |

#### Protocol

```
SuggestedOrder ──▶ RiskEngine ──▶ OrderEvent (to Bus) or nothing

If volume > 0:
    RiskEngine ──publishes──▶ OrderEvent ──▶ Bus
If volume == 0:
    (silent rejection, no event)
```

#### Built-in Implementations

| Config | Behavior |
|---|---|
| `PassthroughRiskConfig` | Always returns original volume (no filtering) |
| Custom (`@strategy.custom_risk_engine`) | User function — can check max drawdown, correlation limits, exposure caps, etc. |

#### Example Custom Risk Behaviors

```python
# Max positions per symbol
def max_positions_risk(suggested_order, modules):
    positions = modules.PORTFOLIO.get_number_of_strategy_open_positions_by_symbol(symbol)
    if positions['TOTAL'] >= 3:
        return 0.0  # reject
    return suggested_order.volume

# Max portfolio exposure
def max_exposure_risk(suggested_order, modules):
    equity = modules.PORTFOLIO.get_account_equity()
    margin_used = equity - modules.PORTFOLIO.get_account_unrealised_pnl()
    if margin_used / equity > 0.5:
        return 0.0  # reject if > 50% margin used
    return suggested_order.volume
```

---

### 29.6 Process 5: Execution Engine (Broker)

This is where the biggest opportunity for multi-provider support exists.

#### Contract

```python
class IExecutionEngine(Protocol):
    # ── Order Execution ──────────────────────────────────────
    def _process_order_event(self, order_event: OrderEvent) -> None:
        """Route an order to the correct execution method."""

    def _send_market_order(self, order_event: OrderEvent) -> OrderSendResult:
        """Execute a market order immediately at current price.
        Must validate margin, create position, and emit FillEvent."""

    def _send_pending_order(self, order_event: OrderEvent) -> OrderSendResult:
        """Place a limit/stop order for future execution.
        Must store the order and check it against price on each bar."""

    # ── Position Management ──────────────────────────────────
    def close_position(self, position_ticket: int) -> OrderSendResult:
        """Close a specific position by ticket. Emit FillEvent(deal=OUT)."""

    def close_all_strategy_positions(self) -> None:
        """Close all positions belonging to this strategy_id."""

    def close_strategy_long_positions_by_symbol(self, symbol: str) -> None:
        """Close all long positions for a symbol under this strategy."""

    def close_strategy_short_positions_by_symbol(self, symbol: str) -> None:
        """Close all short positions for a symbol under this strategy."""

    def update_position_sl_tp(self, ticket: int, new_sl: float, new_tp: float) -> None:
        """Modify SL/TP on an existing position."""

    # ── Order Management ─────────────────────────────────────
    def cancel_pending_order(self, order_ticket: int) -> OrderSendResult:
        """Cancel a pending limit/stop order."""

    def cancel_all_strategy_pending_orders(self) -> None:
        """Cancel all pending orders for this strategy."""

    # ── State Queries ────────────────────────────────────────
    def _get_account_balance(self) -> Decimal:
    def _get_account_equity(self) -> Decimal:
    def _get_account_floating_profit(self) -> Decimal:
    def _get_account_used_margin(self) -> Decimal:
    def _get_account_free_margin(self) -> Decimal:
    def _get_account_currency(self) -> str:
    def _get_strategy_positions(self) -> tuple[OpenPosition]:
    def _get_strategy_pending_orders(self) -> tuple[PendingOrder]:
    def _get_symbol_min_volume(self, symbol: str) -> Decimal:

    # ── Bar-by-Bar Updates ───────────────────────────────────
    def _update_values_and_check_executions_and_fills(self, bar_event: BarEvent) -> None:
        """Called on every base-timeframe bar. Must:
        1. Mark all open positions to market (update unrealized PnL)
        2. Check SL/TP hits → close position → emit FillEvent(deal=OUT)
        3. Check pending order triggers → fill order → emit FillEvent(deal=IN)
        """

    # ── Trading Control ──────────────────────────────────────
    def enable_trading(self) -> None:
    def disable_trading(self) -> None:
```

#### Behavior

| Rule | Description |
|---|---|
| **Margin validation** | Before executing a market order, compute required margin. Reject if free margin is insufficient (margin call). |
| **FillEvent on every execution** | Every successful execution (market fill, pending trigger, SL/TP hit, manual close) must emit a `FillEvent` to the bus. |
| **Deal type semantics** | `FillEvent.deal = DealType.IN` for new positions. `FillEvent.deal = DealType.OUT` for closed positions. |
| **Position tracking** | The engine must track all open positions and pending orders internally. Positions are identified by `ticket` (unique ID). |
| **Strategy isolation** | `strategy_id` (MT5 magic number) scopes positions. Operations like `close_all_strategy_positions()` must only affect positions belonging to that strategy. |
| **SL/TP check order** | On each bar: check SL/TP first (exit), then check pending order triggers (entry). This ensures exits are processed before new entries. |
| **Hedging mode** | Multiple positions in opposite directions on the same symbol are allowed simultaneously. |
| **Slippage model** | Backtest simulator fills at bar open price. Live broker fills at market. The contract does not prescribe slippage — each implementation handles it. |

#### Protocol

```
Bus ──delivers──▶ OrderEvent ──▶ ExecutionEngine
ExecutionEngine ──publishes──▶ FillEvent ──▶ Bus

Also (on each bar, called synchronously by Portfolio):
BarEvent ──▶ ExecutionEngine._update_values_and_check_executions_and_fills()
             ──may publish──▶ FillEvent(deal=OUT) if SL/TP hit
             ──may publish──▶ FillEvent(deal=IN) if pending triggered
```

#### How to Add a New Broker

To add Interactive Brokers, Binance, Alpaca, etc.:

1. Implement `IExecutionEngine` with all methods above.
2. Create a configuration class extending `BaseExecutionConfig`:
   ```python
   class IBKRExecutionConfig(BaseExecutionConfig):
       host: str
       port: int
       client_id: int
       account_id: str
       magic_number: int

   class BinanceExecutionConfig(BaseExecutionConfig):
       api_key: str
       api_secret: str
       testnet: bool = False
       magic_number: int
   ```
3. Implement the broker-specific connector that translates OrderEvents into broker API calls and broker fills into FillEvents.
4. Register in the `ExecutionEngine` service dispatcher.

#### FillEvent Fields (Broker Must Populate)

| Field | Type | Description |
|---|---|---|
| `deal` | DealType | IN (entry) or OUT (exit) |
| `symbol` | str | Instrument traded |
| `position_id` | int | Unique position ticket |
| `strategy_id` | str | Strategy that owns this position |
| `volume` | Decimal | Lots filled |
| `price` | Decimal | Execution price |
| `commission` | Decimal | Broker commission |
| `swap` | Decimal | Overnight swap cost |
| `fee` | Decimal | Exchange fees |
| `gross_profit` | Decimal | Profit on close (0 for entries) |
| `ccy` | str | Currency of costs/profits |

#### Multi-Broker Pattern

To route orders to **different brokers** (e.g., forex to MT5, crypto to Binance):

```python
class RoutingExecutionEngine(IExecutionEngine):
    """Routes orders to the correct broker based on symbol."""

    def __init__(self, routes: dict[str, IExecutionEngine]):
        # routes = {"EURUSD": mt5_engine, "BTCUSDT": binance_engine}
        self.routes = routes

    def _send_market_order(self, order_event):
        broker = self.routes[order_event.symbol]
        return broker._send_market_order(order_event)
```

#### Multi-Account Pattern

To execute the same strategy on **multiple accounts** (e.g., different MT5 accounts):

```python
class MultiAccountExecutionEngine(IExecutionEngine):
    """Fans out orders to multiple accounts."""

    def __init__(self, accounts: list[IExecutionEngine]):
        self.accounts = accounts

    def _send_market_order(self, order_event):
        results = []
        for account in self.accounts:
            results.append(account._send_market_order(order_event))
        return results  # One fill per account
```

Each account would have its own configuration:
```python
accounts = [
    MT5LiveExecutionConfig(magic_number=1001, account="account_A"),
    MT5LiveExecutionConfig(magic_number=1001, account="account_B"),
    BinanceExecutionConfig(api_key="...", magic_number=1001),
]
```

---

### 29.7 Process 6: Portfolio & Trade Archiver

#### Contract — Portfolio

```python
class IPortfolio(Protocol):
    # ── State Updates ────────────────────────────────────────
    def _update_portfolio(self, bar_event: BarEvent) -> None:
        """Recalculate portfolio state on each bar.
        Must call ExecutionEngine to mark positions to market,
        then refresh positions, balance, equity."""

    # ── Position Queries ─────────────────────────────────────
    def get_positions(self, symbol: str = '', ticket: int = None) -> tuple[OpenPosition]:
        """Return open positions, optionally filtered by symbol or ticket."""

    def get_pending_orders(self, symbol: str = '', ticket: int = None) -> tuple[PendingOrder]:
        """Return pending orders, optionally filtered."""

    def get_number_of_strategy_open_positions_by_symbol(self, symbol: str) -> dict:
        """Return {'LONG': int, 'SHORT': int, 'TOTAL': int}."""

    def get_number_of_strategy_pending_orders_by_symbol(self, symbol: str) -> dict:
        """Return {'BUY_LIMIT': int, 'SELL_LIMIT': int, ...}."""

    # ── Account Metrics ──────────────────────────────────────
    def get_account_balance(self) -> Decimal:
    def get_account_equity(self) -> Decimal:
    def get_account_unrealised_pnl(self) -> Decimal:
    def get_account_realised_pnl(self) -> Decimal:

    # ── Historical Data (backtest) ───────────────────────────
    def _export_historical_pnl_dataframe(self) -> pd.DataFrame:
    def _update_portfolio_end_of_backtest(self) -> None:
```

#### Behavior — Portfolio

| Rule | Description |
|---|---|
| **Update only on base timeframe** | `_update_portfolio()` returns early for non-base-timeframe bars to avoid redundant recalculation. |
| **Mark-to-market before anything** | Position PnL must be recalculated before any signal generation happens on that bar. |
| **Historical recording** | In backtest mode, record balance and equity per bar for the first symbol (used for equity curve plotting). |
| **Position source of truth** | Portfolio reads positions from ExecutionEngine. It does not maintain its own position list — it queries the broker layer. |

#### Contract — Trade Archiver

```python
class ITradeArchiver(Protocol):
    def archive_trade(self, fill_event: FillEvent) -> None:
        """Store a fill for historical export."""

    def get_trade_archive(self) -> dict[int, FillEvent]:
        """Return all archived fills keyed by position_id."""

    def export_csv_trade_archive(self, file_path: str) -> None:
        """Export all trades to CSV."""

    def export_historical_trades_dataframe(self) -> pd.DataFrame:
        """Export all trades as a DataFrame."""

    def export_historical_trades_parquet(self, file_path: str) -> None:
        """Export all trades to Parquet."""
```

#### Behavior — Trade Archiver

| Rule | Description |
|---|---|
| **Append-only** | The archiver never modifies or deletes stored trades. |
| **All fills archived** | Both IN and OUT fills must be archived. |
| **Export formats** | Must support at least CSV and DataFrame export. Parquet is optional. |

#### Protocol

```
Bus ──delivers──▶ BarEvent ──▶ Portfolio._update_portfolio()
Bus ──delivers──▶ FillEvent ──▶ TradeArchiver.archive_trade()

Portfolio does not publish events. It is a state container queried by other components.
```

---

### 29.8 Process Support: Schedule Service

#### Contract

```python
class IScheduleService(Protocol):
    def register_callback(
        self, timeframe: StrategyTimeframes, callback: Callable
    ) -> None:
        """Register a function to be called at every boundary of the given timeframe."""

    def run_scheduled_callbacks(self, bar_event: BarEvent) -> None:
        """Check if any timeframe boundaries were crossed since the last bar.
        If so, fire the registered callbacks."""
```

#### Behavior

| Rule | Description |
|---|---|
| **Boundary detection** | A "boundary" means the current bar's timestamp crosses into a new period (e.g., new hour, new day). |
| **Callback signature** | `fn(scheduled_event: ScheduledEvent, modules: Modules)` |
| **Former timestamp tracking** | Each callback tracks when it last fired (`former_execution_timestamp`) so it can detect gaps. |
| **Multiple timeframes** | Multiple callbacks at different timeframes can coexist. Each is checked independently. |

#### Protocol

```
BarEvent ──▶ ScheduleService.run_scheduled_callbacks()
             ──may call──▶ user @run_every callbacks
             ──callbacks may publish──▶ SignalEvent, or call Modules methods
```

---

### 29.9 Process Support: Hook Service

#### Contract

```python
class IHookService(Protocol):
    def register_callback(self, hook: Hooks, callback: Callable) -> None:
        """Register a function to be called at a specific lifecycle point."""

    def call_callbacks(self, hook: Hooks, modules: Modules) -> None:
        """Fire all registered callbacks for the given hook."""
```

#### Hook Points

| Hook | When it fires | Typical use |
|---|---|---|
| `ON_START` | Before the first bar is processed | Initialize state, log start, connect external services |
| `ON_SIGNAL_EVENT` | After a SignalEvent is dequeued, before sizing | Log signals, external alerts |
| `ON_ORDER_EVENT` | After an order is executed | Log trades, send notifications |
| `ON_END` | After the last bar / backtest end | Export results, cleanup, send summary |

---

### 29.10 Portfolio Handler (Orchestrator)

The PortfolioHandler is not a "process" — it is the **orchestrator** that wires sizing → risk → execution within a single process. In a distributed architecture, its role is replaced by the message broker routing.

#### Contract

```python
class IPortfolioHandler(Protocol):
    def process_bar_event(self, bar_event: BarEvent) -> None:
        """Handle a BAR event: update portfolio, then pass to schedule + signal."""

    def process_signal_event(self, signal_event: SignalEvent) -> None:
        """Handle a SIGNAL event: size it, risk-check it, emit ORDER if approved."""

    def process_fill_event(self, fill_event: FillEvent) -> None:
        """Handle a FILL event: archive the trade."""

    def process_backtest_end(self, name: str, export_csv: bool, export_parquet: bool) -> BacktestResults:
        """Finalize backtest: close positions, export data, return results."""
```

#### Behavior

| Rule | Description |
|---|---|
| **BAR processing order** | 1) Update portfolio → 2) Run scheduled callbacks → 3) Generate signals. This order is invariant. |
| **SIGNAL processing chain** | 1) Size → 2) Risk check → 3) Emit ORDER (if approved). This is synchronous within one handler. |
| **FILL processing** | Delegate to TradeArchiver. No other logic. |

---

### 29.11 Contract Compliance Checklist

Use this when implementing a new provider or broker. Every item must be satisfied for the component to work correctly in the pipeline.

#### New Data Provider Checklist

- [ ] Implements all `IDataProvider` methods
- [ ] Bars emitted in chronological order
- [ ] Multi-symbol bars aligned on same timestamps
- [ ] Higher timeframe bars emitted after base timeframe
- [ ] `get_latest_bars()` returns only completed bars (no lookahead)
- [ ] Prices encoded as integers with correct `digits` field
- [ ] End-of-data signaled cleanly
- [ ] Configuration class extends `BaseDataConfig`
- [ ] Registered in DataProvider service dispatcher

#### New Broker Checklist

- [ ] Implements all `IExecutionEngine` methods
- [ ] Validates margin before execution
- [ ] Emits `FillEvent(deal=IN)` on every successful entry
- [ ] Emits `FillEvent(deal=OUT)` on every exit (close, SL, TP)
- [ ] Populates all FillEvent fields (commission, swap, fee, gross_profit)
- [ ] Tracks positions by `ticket` (unique ID)
- [ ] Scopes operations by `strategy_id`
- [ ] Checks SL/TP on every bar (via `_update_values_and_check_executions_and_fills`)
- [ ] Checks pending order triggers on every bar
- [ ] Supports hedging mode (multiple positions, both directions)
- [ ] Configuration class extends `BaseExecutionConfig`
- [ ] Registered in ExecutionEngine service dispatcher

#### New Signal Engine Checklist

- [ ] Implements `generate_signal(bar_event, modules) → SignalEvent | list | None`
- [ ] Filters on correct timeframe
- [ ] Sets `strategy_id` on every SignalEvent
- [ ] Sets `sl != 0` if RiskPctSizing is used downstream
- [ ] Deterministic on replay (same bars → same signals)

#### New Sizing Engine Checklist

- [ ] Implements `get_suggested_order(signal_event, modules) → SuggestedOrder`
- [ ] Respects symbol volume limits (min, max, step)
- [ ] Returns volume = 0 if unable to size
- [ ] No side effects (read-only)

#### New Risk Engine Checklist

- [ ] Implements `assess_order(suggested_order, modules) → float`
- [ ] Returns volume > 0 to approve, 0 to reject
- [ ] No side effects (read-only)

---

### 29.12 Full Protocol Map

Summary of all events flowing between all components:

```
                          ┌──────────────────────────────────────┐
                          │           MESSAGE BUS                │
                          │                                      │
  DataProvider ──────────▶│  BarEvent                            │
                          │    │                                 │
                          │    ├──▶ Portfolio._update_portfolio() │
                          │    ├──▶ ScheduleService              │
                          │    └──▶ SignalEngine                 │
                          │                                      │
  SignalEngine ──────────▶│  SignalEvent                         │
                          │    └──▶ SizingEngine                 │
                          │                                      │
  SizingEngine ──────────▶│  SuggestedOrder(Event)               │
                          │    └──▶ RiskEngine                   │
                          │                                      │
  RiskEngine ────────────▶│  OrderEvent                          │
                          │    └──▶ ExecutionEngine               │
                          │                                      │
  ExecutionEngine ───────▶│  FillEvent                           │
                          │    └──▶ Portfolio + TradeArchiver     │
                          │                                      │
  ExecutionEngine ───────▶│  AccountStateEvent (distributed only)│
                          │    └──▶ All components (local cache)  │
                          └──────────────────────────────────────┘
```

Each arrow represents a contract boundary. As long as the sender produces the correct event shape and the receiver handles it according to its behavioral rules, the two sides can be implemented independently, in any language, running in any process.

---
