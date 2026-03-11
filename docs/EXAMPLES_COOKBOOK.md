# PyEventBT — Examples & Cookbook

Practical examples for daily trading tasks using pyeventbt.

---

## Table of Contents

1. [Installation & Setup](#1-installation--setup)
2. [Minimal Strategy (Hello World)](#2-minimal-strategy-hello-world)
3. [RSI Mean Reversion](#3-rsi-mean-reversion)
4. [Multi-Symbol Portfolio](#4-multi-symbol-portfolio)
5. [Multi-Timeframe Strategy](#5-multi-timeframe-strategy)
6. [Custom Sizing: Risk % per Trade](#6-custom-sizing-risk--per-trade)
7. [Custom Risk Engine: Max Positions](#7-custom-risk-engine-max-positions)
8. [Using Hooks (Logging, Notifications)](#8-using-hooks-logging-notifications)
9. [Scheduled Tasks (@run_every)](#9-scheduled-tasks-run_every)
10. [Stop Loss & Take Profit](#10-stop-loss--take-profit)
11. [Pending Orders (LIMIT & STOP)](#11-pending-orders-limit--stop)
12. [Managing Positions at Runtime](#12-managing-positions-at-runtime)
13. [Accessing Portfolio State](#13-accessing-portfolio-state)
14. [Downloading Data with Quantdle](#14-downloading-data-with-quantdle)
15. [Analyzing Backtest Results](#15-analyzing-backtest-results)
16. [Going Live with MT5](#16-going-live-with-mt5)
17. [Full Strategy Template](#17-full-strategy-template)

---

## 1. Installation & Setup

```bash
# Install with Poetry
pip install pyeventbt

# Or clone and install locally
git clone https://github.com/marticastany/pyeventbt.git
cd pyeventbt
poetry install
```

**CSV data format** (MT5 export style):
```
date	time	open	high	low	close	tickvol	volume	spread
2020.01.02	00:00:00	1.12130	1.12140	1.12100	1.12110	150	0	12
2020.01.02	00:01:00	1.12110	1.12130	1.12090	1.12100	120	0	12
```

Place CSV files as `{symbol}.csv` (e.g., `EURUSD.csv`) in your data directory.

---

## 2. Minimal Strategy (Hello World)

The simplest possible strategy: buy whenever a daily bar closes.

```python
from pyeventbt import (
    Strategy, BarEvent, SignalEvent, Modules,
    StrategyTimeframes, PassthroughRiskConfig, MinSizingConfig,
)
from datetime import datetime
from decimal import Decimal

strategy = Strategy()
strategy_id = "100001"
tf = StrategyTimeframes.ONE_DAY

@strategy.custom_signal_engine(strategy_id=strategy_id, strategy_timeframes=[tf])
def my_signal(event: BarEvent, modules: Modules):
    if event.timeframe != tf:
        return

    symbol = event.symbol
    positions = modules.PORTFOLIO.get_number_of_strategy_open_positions_by_symbol(symbol)

    # Only buy if we have no position
    if positions['TOTAL'] > 0:
        return

    # Time for next bar (avoid lookahead)
    if modules.TRADING_CONTEXT == "BACKTEST":
        time_generated = event.datetime + tf.to_timedelta()
    else:
        time_generated = datetime.now()

    tick = modules.DATA_PROVIDER.get_latest_tick(symbol)

    return SignalEvent(
        symbol=symbol,
        time_generated=time_generated,
        strategy_id=strategy_id,
        signal_type="BUY",
        order_type="MARKET",
        order_price=tick['ask'],
        sl=Decimal("0"),
        tp=Decimal("0"),
    )

# Configure engines
strategy.configure_predefined_sizing_engine(MinSizingConfig())
strategy.configure_predefined_risk_engine(PassthroughRiskConfig())

# Run backtest
backtest = strategy.backtest(
    strategy_id=strategy_id,
    initial_capital=10000,
    symbols_to_trade=["EURUSD"],
    csv_dir="/path/to/your/csv/data",
    start_date=datetime(2022, 1, 1),
    end_date=datetime(2023, 1, 1),
)

backtest.plot()
```

---

## 3. RSI Mean Reversion

Buy when RSI < 30 (oversold), sell when RSI > 70 (overbought).

```python
from pyeventbt import (
    Strategy, BarEvent, SignalEvent, Modules,
    StrategyTimeframes, PassthroughRiskConfig, MinSizingConfig,
)
from pyeventbt.indicators import RSI
from datetime import datetime
from decimal import Decimal

strategy = Strategy()
strategy_id = "200001"
tf = StrategyTimeframes.ONE_HOUR

# Parameters
rsi_period = 14
rsi_oversold = 30
rsi_overbought = 70

@strategy.custom_signal_engine(strategy_id=strategy_id, strategy_timeframes=[tf])
def rsi_mean_reversion(event: BarEvent, modules: Modules):
    if event.timeframe != tf:
        return

    symbol = event.symbol
    bars = modules.DATA_PROVIDER.get_latest_bars(symbol, tf, rsi_period + 10)

    if bars is None or bars.height < rsi_period + 5:
        return

    close = bars.select('close').to_numpy().flatten()
    rsi_values = RSI.compute(close, rsi_period)
    current_rsi = rsi_values[-1]

    positions = modules.PORTFOLIO.get_number_of_strategy_open_positions_by_symbol(symbol)

    if modules.TRADING_CONTEXT == "BACKTEST":
        time_gen = event.datetime + tf.to_timedelta()
    else:
        time_gen = datetime.now()

    tick = modules.DATA_PROVIDER.get_latest_tick(symbol)

    # Oversold → BUY
    if current_rsi < rsi_oversold and positions['LONG'] == 0:
        if positions['SHORT'] > 0:
            modules.EXECUTION_ENGINE.close_strategy_short_positions_by_symbol(symbol)
        return SignalEvent(
            symbol=symbol,
            time_generated=time_gen,
            strategy_id=strategy_id,
            signal_type="BUY",
            order_type="MARKET",
            order_price=tick['ask'],
            sl=Decimal("0"), tp=Decimal("0"),
        )

    # Overbought → SELL
    if current_rsi > rsi_overbought and positions['SHORT'] == 0:
        if positions['LONG'] > 0:
            modules.EXECUTION_ENGINE.close_strategy_long_positions_by_symbol(symbol)
        return SignalEvent(
            symbol=symbol,
            time_generated=time_gen,
            strategy_id=strategy_id,
            signal_type="SELL",
            order_type="MARKET",
            order_price=tick['bid'],
            sl=Decimal("0"), tp=Decimal("0"),
        )

strategy.configure_predefined_sizing_engine(MinSizingConfig())
strategy.configure_predefined_risk_engine(PassthroughRiskConfig())

backtest = strategy.backtest(
    strategy_id=strategy_id,
    initial_capital=50000,
    symbols_to_trade=["EURUSD"],
    csv_dir="/path/to/csv",
    start_date=datetime(2022, 1, 1),
    end_date=datetime(2023, 12, 1),
)
backtest.plot()
```

---

## 4. Multi-Symbol Portfolio

Trade multiple symbols with the same strategy logic.

```python
from pyeventbt import (
    Strategy, BarEvent, SignalEvent, Modules,
    StrategyTimeframes, PassthroughRiskConfig, MinSizingConfig,
)
from pyeventbt.indicators import SMA
from datetime import datetime
from decimal import Decimal

strategy = Strategy()
strategy_id = "300001"
tf = StrategyTimeframes.ONE_DAY

# Trade 3 FX pairs simultaneously
symbols_to_trade = ["EURUSD", "GBPUSD", "USDJPY"]

fast_period = 10
slow_period = 30

@strategy.custom_signal_engine(strategy_id=strategy_id, strategy_timeframes=[tf])
def multi_symbol_ma(event: BarEvent, modules: Modules):
    if event.timeframe != tf:
        return

    symbol = event.symbol  # The engine is called once PER symbol PER bar
    bars = modules.DATA_PROVIDER.get_latest_bars(symbol, tf, slow_period + 10)

    if bars is None or bars.height < slow_period + 5:
        return

    close = bars.select('close').to_numpy().flatten()
    fast_ma = SMA.compute(close, fast_period)[-1]
    slow_ma = SMA.compute(close, slow_period)[-1]

    positions = modules.PORTFOLIO.get_number_of_strategy_open_positions_by_symbol(symbol)

    if modules.TRADING_CONTEXT == "BACKTEST":
        time_gen = event.datetime + tf.to_timedelta()
    else:
        time_gen = datetime.now()

    tick = modules.DATA_PROVIDER.get_latest_tick(symbol)

    if fast_ma > slow_ma and positions['LONG'] == 0:
        if positions['SHORT'] > 0:
            modules.EXECUTION_ENGINE.close_strategy_short_positions_by_symbol(symbol)
        return SignalEvent(
            symbol=symbol, time_generated=time_gen, strategy_id=strategy_id,
            signal_type="BUY", order_type="MARKET",
            order_price=tick['ask'], sl=Decimal("0"), tp=Decimal("0"),
        )

    if fast_ma < slow_ma and positions['SHORT'] == 0:
        if positions['LONG'] > 0:
            modules.EXECUTION_ENGINE.close_strategy_long_positions_by_symbol(symbol)
        return SignalEvent(
            symbol=symbol, time_generated=time_gen, strategy_id=strategy_id,
            signal_type="SELL", order_type="MARKET",
            order_price=tick['bid'], sl=Decimal("0"), tp=Decimal("0"),
        )

strategy.configure_predefined_sizing_engine(MinSizingConfig())
strategy.configure_predefined_risk_engine(PassthroughRiskConfig())

backtest = strategy.backtest(
    strategy_id=strategy_id,
    initial_capital=100000,
    symbols_to_trade=symbols_to_trade,  # <-- pass all symbols here
    csv_dir="/path/to/csv",  # needs EURUSD.csv, GBPUSD.csv, USDJPY.csv
    start_date=datetime(2022, 1, 1),
    end_date=datetime(2023, 12, 1),
)
backtest.plot()
```

---

## 5. Multi-Timeframe Strategy

Use daily bars for trend direction + hourly bars for entry timing.

```python
from pyeventbt import (
    Strategy, BarEvent, SignalEvent, Modules,
    StrategyTimeframes, PassthroughRiskConfig, MinSizingConfig,
)
from pyeventbt.indicators import SMA, RSI
from datetime import datetime
from decimal import Decimal

strategy = Strategy()
strategy_id = "400001"

daily_tf = StrategyTimeframes.ONE_DAY
hourly_tf = StrategyTimeframes.ONE_HOUR

@strategy.custom_signal_engine(
    strategy_id=strategy_id,
    strategy_timeframes=[hourly_tf, daily_tf]  # subscribe to both
)
def multi_tf_strategy(event: BarEvent, modules: Modules):
    # Only generate signals on hourly bars
    if event.timeframe != hourly_tf:
        return

    symbol = event.symbol

    # 1. Get DAILY trend direction (higher timeframe filter)
    daily_bars = modules.DATA_PROVIDER.get_latest_bars(symbol, daily_tf, 50)
    if daily_bars is None or daily_bars.height < 30:
        return

    daily_close = daily_bars.select('close').to_numpy().flatten()
    daily_sma_fast = SMA.compute(daily_close, 10)[-1]
    daily_sma_slow = SMA.compute(daily_close, 30)[-1]

    if daily_sma_fast > daily_sma_slow:
        trend = "UP"
    elif daily_sma_fast < daily_sma_slow:
        trend = "DOWN"
    else:
        return

    # 2. Get HOURLY RSI for entry timing
    hourly_bars = modules.DATA_PROVIDER.get_latest_bars(symbol, hourly_tf, 30)
    if hourly_bars is None or hourly_bars.height < 20:
        return

    hourly_close = hourly_bars.select('close').to_numpy().flatten()
    rsi = RSI.compute(hourly_close, 14)[-1]

    positions = modules.PORTFOLIO.get_number_of_strategy_open_positions_by_symbol(symbol)

    if modules.TRADING_CONTEXT == "BACKTEST":
        time_gen = event.datetime + hourly_tf.to_timedelta()
    else:
        time_gen = datetime.now()

    tick = modules.DATA_PROVIDER.get_latest_tick(symbol)

    # Buy dips in uptrend
    if trend == "UP" and rsi < 35 and positions['LONG'] == 0:
        if positions['SHORT'] > 0:
            modules.EXECUTION_ENGINE.close_strategy_short_positions_by_symbol(symbol)
        return SignalEvent(
            symbol=symbol, time_generated=time_gen, strategy_id=strategy_id,
            signal_type="BUY", order_type="MARKET",
            order_price=tick['ask'], sl=Decimal("0"), tp=Decimal("0"),
        )

    # Sell rallies in downtrend
    if trend == "DOWN" and rsi > 65 and positions['SHORT'] == 0:
        if positions['LONG'] > 0:
            modules.EXECUTION_ENGINE.close_strategy_long_positions_by_symbol(symbol)
        return SignalEvent(
            symbol=symbol, time_generated=time_gen, strategy_id=strategy_id,
            signal_type="SELL", order_type="MARKET",
            order_price=tick['bid'], sl=Decimal("0"), tp=Decimal("0"),
        )

strategy.configure_predefined_sizing_engine(MinSizingConfig())
strategy.configure_predefined_risk_engine(PassthroughRiskConfig())

backtest = strategy.backtest(
    strategy_id=strategy_id,
    initial_capital=50000,
    symbols_to_trade=["EURUSD"],
    csv_dir="/path/to/csv",
    start_date=datetime(2022, 1, 1),
    end_date=datetime(2023, 12, 1),
)
backtest.plot()
```

---

## 6. Custom Sizing: Risk % per Trade

Use the built-in `RiskPctSizingConfig` or write your own sizing engine.

### Option A: Built-in Risk Percentage Sizing

```python
from pyeventbt import RiskPctSizingConfig

# Risk 1% of account equity per trade (requires SL on signals)
strategy.configure_predefined_sizing_engine(
    RiskPctSizingConfig(risk_pct=1.0)
)
```

### Option B: Built-in Fixed Sizing

```python
from pyeventbt import FixedSizingConfig
from decimal import Decimal

# Always trade 0.5 lots
strategy.configure_predefined_sizing_engine(
    FixedSizingConfig(volume=Decimal("0.5"))
)
```

### Option C: Fully Custom Sizing Engine

```python
from pyeventbt import Strategy, Modules, SignalEvent
from pyeventbt.portfolio_handler.core.entities.suggested_order import SuggestedOrder
from decimal import Decimal

strategy = Strategy()
strategy_id = "500001"

@strategy.custom_sizing_engine(strategy_id=strategy_id)
def my_sizing(signal_event: SignalEvent, modules: Modules) -> SuggestedOrder:
    """Size based on account equity: 2% of equity converted to lots."""
    equity = modules.EXECUTION_ENGINE._get_account_equity()

    # Simple example: 0.01 lot per $1000 of equity
    volume = Decimal(str(round(float(equity) / 1000 * 0.01, 2)))
    volume = max(volume, Decimal("0.01"))  # ensure minimum

    return SuggestedOrder(
        signal_event=signal_event,
        volume=volume,
    )
```

---

## 7. Custom Risk Engine: Max Positions

Limit the number of simultaneous open positions.

```python
from pyeventbt import Strategy, Modules
from pyeventbt.portfolio_handler.core.entities.suggested_order import SuggestedOrder
from decimal import Decimal

strategy = Strategy()
strategy_id = "600001"
max_open_positions = 3

@strategy.custom_risk_engine(strategy_id=strategy_id)
def max_positions_risk(suggested_order: SuggestedOrder, modules: Modules) -> float:
    """Reject orders if we already have too many open positions."""
    symbol = suggested_order.signal_event.symbol
    positions = modules.PORTFOLIO.get_number_of_strategy_open_positions_by_symbol(symbol)

    if positions['TOTAL'] >= max_open_positions:
        return 0.0  # Reject — volume 0 means order is discarded

    return float(suggested_order.volume)  # Approve — pass volume through
```

---

## 8. Using Hooks (Logging, Notifications)

Hooks let you run code at key lifecycle moments.

```python
from pyeventbt import Strategy, Modules
from pyeventbt.hooks.hook_service import Hooks

strategy = Strategy()

@strategy.hook(Hooks.ON_START)
def on_start(modules: Modules):
    balance = modules.PORTFOLIO.get_account_balance()
    print(f"Strategy started! Initial balance: {balance}")

@strategy.hook(Hooks.ON_SIGNAL_EVENT)
def on_signal(modules: Modules):
    print("Signal generated!")

@strategy.hook(Hooks.ON_ORDER_EVENT)
def on_order(modules: Modules):
    equity = modules.PORTFOLIO.get_account_equity()
    print(f"Order executed! Current equity: {equity}")

@strategy.hook(Hooks.ON_END)
def on_end(modules: Modules):
    balance = modules.PORTFOLIO.get_account_balance()
    print(f"Strategy finished! Final balance: {balance}")

# You can also toggle hooks on/off at runtime
# strategy.disable_hooks()
# strategy.enable_hooks()
```

---

## 9. Scheduled Tasks (@run_every)

Execute code on a fixed schedule (e.g., rebalance weekly, log daily).

```python
from pyeventbt import Strategy, Modules, StrategyTimeframes
from pyeventbt.events.events import ScheduledEvent

strategy = Strategy()

@strategy.run_every(StrategyTimeframes.ONE_DAY)
def daily_report(event: ScheduledEvent, modules: Modules):
    """Print a daily equity snapshot."""
    balance = modules.PORTFOLIO.get_account_balance()
    equity = modules.PORTFOLIO.get_account_equity()
    print(f"[{event.timestamp}] Balance: {balance} | Equity: {equity}")

@strategy.run_every(StrategyTimeframes.ONE_WEEK)
def weekly_rebalance(event: ScheduledEvent, modules: Modules):
    """Close all positions every week for rebalancing."""
    print(f"[{event.timestamp}] Weekly rebalance — closing all positions")
    modules.EXECUTION_ENGINE.close_all_strategy_positions()

# IMPORTANT: Enable schedules when running backtest
backtest = strategy.backtest(
    strategy_id="700001",
    initial_capital=50000,
    symbols_to_trade=["EURUSD"],
    csv_dir="/path/to/csv",
    start_date=datetime(2022, 1, 1),
    end_date=datetime(2023, 1, 1),
    run_scheduled_taks=True,  # <-- required to enable @run_every
)
```

---

## 10. Stop Loss & Take Profit

Set SL/TP on your signals for automatic exit.

```python
@strategy.custom_signal_engine(strategy_id=strategy_id, strategy_timeframes=[tf])
def strategy_with_sl_tp(event: BarEvent, modules: Modules):
    if event.timeframe != tf:
        return

    symbol = event.symbol
    tick = modules.DATA_PROVIDER.get_latest_tick(symbol)
    ask = tick['ask']
    bid = tick['bid']

    positions = modules.PORTFOLIO.get_number_of_strategy_open_positions_by_symbol(symbol)
    if positions['TOTAL'] > 0:
        return

    if modules.TRADING_CONTEXT == "BACKTEST":
        time_gen = event.datetime + tf.to_timedelta()
    else:
        time_gen = datetime.now()

    # BUY with 50 pip SL and 100 pip TP
    sl_distance = Decimal("0.0050")  # 50 pips
    tp_distance = Decimal("0.0100")  # 100 pips

    return SignalEvent(
        symbol=symbol,
        time_generated=time_gen,
        strategy_id=strategy_id,
        signal_type="BUY",
        order_type="MARKET",
        order_price=ask,
        sl=ask - sl_distance,     # Stop Loss below entry
        tp=ask + tp_distance,     # Take Profit above entry
    )
```

You can also **update SL/TP on existing positions** at runtime:

```python
# Move stop loss to break-even on an existing position
positions = modules.PORTFOLIO.get_positions(symbol="EURUSD")
for pos in positions:
    if float(pos.unrealized_profit) > 0:
        # Trailing stop to entry price (break-even)
        modules.EXECUTION_ENGINE.update_position_sl_tp(
            position_ticket=pos.ticket,
            new_sl=float(pos.price_entry),
            new_tp=0.0,  # keep original TP
        )
```

---

## 11. Pending Orders (LIMIT & STOP)

Place orders that trigger at a future price.

```python
@strategy.custom_signal_engine(strategy_id=strategy_id, strategy_timeframes=[tf])
def pending_order_strategy(event: BarEvent, modules: Modules):
    if event.timeframe != tf:
        return

    symbol = event.symbol
    pending = modules.PORTFOLIO.get_number_of_strategy_pending_orders_by_symbol(symbol)

    # Only place if no pending orders exist
    if pending['TOTAL'] > 0:
        return

    if modules.TRADING_CONTEXT == "BACKTEST":
        time_gen = event.datetime + tf.to_timedelta()
    else:
        time_gen = datetime.now()

    tick = modules.DATA_PROVIDER.get_latest_tick(symbol)
    current_price = tick['ask']

    signals = []

    # BUY STOP: buy if price breaks above current + 30 pips
    signals.append(SignalEvent(
        symbol=symbol,
        time_generated=time_gen,
        strategy_id=strategy_id,
        signal_type="BUY",
        order_type="STOP",  # triggers when price reaches order_price
        order_price=current_price + Decimal("0.0030"),
        sl=Decimal("0"), tp=Decimal("0"),
    ))

    # SELL LIMIT: sell at a better price (above current)
    signals.append(SignalEvent(
        symbol=symbol,
        time_generated=time_gen,
        strategy_id=strategy_id,
        signal_type="SELL",
        order_type="LIMIT",  # triggers when price reaches order_price
        order_price=current_price + Decimal("0.0050"),
        sl=Decimal("0"), tp=Decimal("0"),
    ))

    return signals
```

**Cancel pending orders:**
```python
# Cancel all pending orders
modules.EXECUTION_ENGINE.cancel_all_strategy_pending_orders()

# Cancel only BUY_LIMIT orders for EURUSD
modules.EXECUTION_ENGINE.cancel_all_strategy_pending_orders_by_type_and_symbol("BUY_LIMIT", "EURUSD")
```

---

## 12. Managing Positions at Runtime

Close positions selectively from within your signal engine or hooks.

```python
# Close ALL positions
modules.EXECUTION_ENGINE.close_all_strategy_positions()

# Close only LONG positions for a symbol
modules.EXECUTION_ENGINE.close_strategy_long_positions_by_symbol("EURUSD")

# Close only SHORT positions for a symbol
modules.EXECUTION_ENGINE.close_strategy_short_positions_by_symbol("EURUSD")

# Close a specific position by ticket number
positions = modules.PORTFOLIO.get_positions(symbol="EURUSD")
for pos in positions:
    if some_condition:
        modules.EXECUTION_ENGINE.close_position(pos.ticket)

# Enable/disable trading (blocks all order execution)
modules.EXECUTION_ENGINE.disable_trading()
modules.EXECUTION_ENGINE.enable_trading()
```

---

## 13. Accessing Portfolio State

Read account and position data anywhere you have `modules`.

```python
# Account info
balance = modules.PORTFOLIO.get_account_balance()       # Decimal
equity = modules.PORTFOLIO.get_account_equity()         # Decimal
unrealized = modules.PORTFOLIO.get_account_unrealised_pnl()
realized = modules.PORTFOLIO.get_account_realised_pnl()

# Position counts by symbol
pos_info = modules.PORTFOLIO.get_number_of_strategy_open_positions_by_symbol("EURUSD")
# Returns: {"LONG": 1, "SHORT": 0, "TOTAL": 1}

# Pending order counts
ord_info = modules.PORTFOLIO.get_number_of_strategy_pending_orders_by_symbol("EURUSD")
# Returns: {"BUY_LIMIT": 0, "SELL_LIMIT": 1, "BUY_STOP": 0, "SELL_STOP": 0, "TOTAL": 1}

# Detailed position objects
positions = modules.PORTFOLIO.get_positions(symbol="EURUSD")
for pos in positions:
    print(f"  Ticket: {pos.ticket}")
    print(f"  Type: {pos.type}")           # "BUY" or "SELL"
    print(f"  Entry: {pos.price_entry}")
    print(f"  Volume: {pos.volume}")
    print(f"  P&L: {pos.unrealized_profit}")
    print(f"  SL: {pos.sl}, TP: {pos.tp}")

# Detailed pending order objects
pending = modules.PORTFOLIO.get_pending_orders(symbol="EURUSD")
for order in pending:
    print(f"  Ticket: {order.ticket}")
    print(f"  Type: {order.type}")         # "BUY_LIMIT", "SELL_STOP", etc.
    print(f"  Price: {order.price}")
    print(f"  Volume: {order.volume}")

# Data provider access
bars = modules.DATA_PROVIDER.get_latest_bars("EURUSD", StrategyTimeframes.ONE_HOUR, 100)
# Returns: polars DataFrame with columns: datetime, open, high, low, close, tickvol, volume, spread

tick = modules.DATA_PROVIDER.get_latest_tick("EURUSD")
# Returns: dict with 'bid', 'ask', etc.

bid = modules.DATA_PROVIDER.get_latest_bid("EURUSD")  # Decimal
ask = modules.DATA_PROVIDER.get_latest_ask("EURUSD")  # Decimal
```

---

## 14. Downloading Data with Quantdle

Automatically fetch and cache historical data.

```python
from pyeventbt import QuantdleDataUpdater
from datetime import datetime

updater = QuantdleDataUpdater(
    api_key="your_api_key",
    api_key_id="your_api_key_id",
)

# Downloads data and saves as CSV (smart caching — only fetches missing ranges)
updater.update_data(
    csv_dir="/path/to/csv/cache",
    symbols=["EURUSD", "GBPUSD"],
    start_date=datetime(2020, 1, 1),
    end_date=datetime(2024, 1, 1),
    timeframe="1min",  # base timeframe (resampled by the engine)
)

# Now use the same csv_dir in your backtest
backtest = strategy.backtest(
    csv_dir="/path/to/csv/cache",
    # ...
)
```

---

## 15. Analyzing Backtest Results

```python
backtest = strategy.backtest(...)

# Plot equity and balance curves
backtest.plot()

# Access raw data
pnl_df = backtest.pnl        # pandas DataFrame: EQUITY, BALANCE columns
returns = backtest.returns    # pandas Series: % change in equity
trades_df = backtest.trades   # pandas DataFrame: all closed trades

# Quick stats you can compute
import numpy as np

total_return = (pnl_df['EQUITY'].iloc[-1] / pnl_df['EQUITY'].iloc[0] - 1) * 100
max_equity = pnl_df['EQUITY'].cummax()
drawdown = (pnl_df['EQUITY'] - max_equity) / max_equity * 100
max_drawdown = drawdown.min()
sharpe = returns.mean() / returns.std() * np.sqrt(252) if returns.std() > 0 else 0

print(f"Total Return: {total_return:.2f}%")
print(f"Max Drawdown: {max_drawdown:.2f}%")
print(f"Sharpe Ratio: {sharpe:.2f}")
print(f"Total Trades: {len(trades_df)}")

# Export results
backtest = strategy.backtest(
    export_backtest_csv=True,       # saves to Desktop/PyEventBT/
    export_backtest_parquet=True,   # zstd compressed
    backtest_results_dir="/custom/path",  # optional custom directory
    # ...
)
```

---

## 16. Going Live with MT5

Switch from backtest to live with minimal changes. **Windows only.**

```python
from pyeventbt import Strategy, Mt5PlatformConfig

strategy = Strategy()
strategy_id = "123456"

# ... define your signal/sizing/risk engines (same as backtest) ...

# MT5 connection config
mt5_config = Mt5PlatformConfig(
    path=r"C:\Program Files\MetaTrader 5\terminal64.exe",
    login=12345678,
    password="your_password",
    server="YourBroker-Demo",
    timeout=60000,
    portable=False,
)

# Start live trading (infinite loop)
strategy.run_live(
    mt5_configuration=mt5_config,
    strategy_id=strategy_id,
    initial_capital=100000,
    symbols_to_trade=["EURUSD", "GBPUSD"],
    heartbeat=0.1,  # seconds between queue polls
)
```

**Key differences from backtest:**
- `time_generated` should use `datetime.now()` instead of `event.datetime + timedelta`
- Data comes from MT5 API instead of CSV files
- Orders are placed on the real market via `mt5.order_send()`
- Use `modules.TRADING_CONTEXT` to branch behavior:

```python
if modules.TRADING_CONTEXT == "BACKTEST":
    time_gen = event.datetime + tf.to_timedelta()
else:
    time_gen = datetime.now()
```

---

## 17. Full Strategy Template

Copy-paste this as a starting point for any new strategy.

```python
"""
My Strategy — [describe what it does]
"""
from pyeventbt import (
    Strategy, BarEvent, SignalEvent, Modules,
    StrategyTimeframes, PassthroughRiskConfig, MinSizingConfig,
)
from pyeventbt.hooks.hook_service import Hooks
from pyeventbt.events.events import ScheduledEvent
from pyeventbt.indicators import SMA, RSI, EMA, ATR
from datetime import datetime
from decimal import Decimal
import logging

# ─── CONFIGURATION ───────────────────────────────────────────
strategy = Strategy(logging_level=logging.INFO)
strategy_id = "123456"  # must be digits (maps to MT5 Magic Number)

tf = StrategyTimeframes.ONE_HOUR
symbols_to_trade = ["EURUSD"]
initial_capital = 50000

# ─── STRATEGY PARAMETERS ────────────────────────────────────
# Define your indicator parameters here
param_1 = 14
param_2 = 30


# ─── SIGNAL ENGINE ──────────────────────────────────────────
@strategy.custom_signal_engine(strategy_id=strategy_id, strategy_timeframes=[tf])
def my_signal_engine(event: BarEvent, modules: Modules):
    if event.timeframe != tf:
        return

    symbol = event.symbol

    # 1. Get data
    bars = modules.DATA_PROVIDER.get_latest_bars(symbol, tf, param_2 + 10)
    if bars is None or bars.height < param_2 + 5:
        return

    close = bars.select('close').to_numpy().flatten()

    # 2. Calculate indicators
    # indicator_value = SMA.compute(close, param_1)[-1]

    # 3. Check current positions
    positions = modules.PORTFOLIO.get_number_of_strategy_open_positions_by_symbol(symbol)

    # 4. Generate signal time
    if modules.TRADING_CONTEXT == "BACKTEST":
        time_gen = event.datetime + tf.to_timedelta()
    else:
        time_gen = datetime.now()

    tick = modules.DATA_PROVIDER.get_latest_tick(symbol)

    # 5. Signal logic
    # if buy_condition and positions['LONG'] == 0:
    #     if positions['SHORT'] > 0:
    #         modules.EXECUTION_ENGINE.close_strategy_short_positions_by_symbol(symbol)
    #     return SignalEvent(
    #         symbol=symbol, time_generated=time_gen, strategy_id=strategy_id,
    #         signal_type="BUY", order_type="MARKET",
    #         order_price=tick['ask'],
    #         sl=Decimal("0"), tp=Decimal("0"),
    #     )

    return None


# ─── HOOKS (optional) ───────────────────────────────────────
@strategy.hook(Hooks.ON_START)
def on_start(modules: Modules):
    print(f"Started with balance: {modules.PORTFOLIO.get_account_balance()}")

@strategy.hook(Hooks.ON_END)
def on_end(modules: Modules):
    print(f"Finished with balance: {modules.PORTFOLIO.get_account_balance()}")


# ─── SCHEDULED TASKS (optional) ─────────────────────────────
# @strategy.run_every(StrategyTimeframes.ONE_DAY)
# def daily_task(event: ScheduledEvent, modules: Modules):
#     print(f"Daily check at {event.timestamp}")


# ─── ENGINE CONFIGURATION ───────────────────────────────────
strategy.configure_predefined_sizing_engine(MinSizingConfig())
strategy.configure_predefined_risk_engine(PassthroughRiskConfig())


# ─── RUN BACKTEST ────────────────────────────────────────────
if __name__ == "__main__":
    backtest = strategy.backtest(
        strategy_id=strategy_id,
        initial_capital=initial_capital,
        symbols_to_trade=symbols_to_trade,
        csv_dir="/path/to/csv/data",
        start_date=datetime(2022, 1, 1),
        end_date=datetime(2023, 12, 1),
        backtest_name="my_strategy",
        account_currency="USD",
        account_leverage=30,
        export_backtest_parquet=True,
        run_scheduled_taks=False,  # set True if using @run_every
    )

    backtest.plot()

    # Quick stats
    pnl = backtest.pnl
    print(f"Final equity: {pnl['EQUITY'].iloc[-1]:.2f}")
    print(f"Total trades: {len(backtest.trades)}")
```

---

## Quick Reference: Common Patterns

| Task | Code |
|---|---|
| Get close prices as numpy | `bars.select('close').to_numpy().flatten()` |
| Get OHLC as numpy | `bars.select(['open','high','low','close']).to_numpy()` |
| Check if in backtest | `modules.TRADING_CONTEXT == "BACKTEST"` |
| Get position count | `modules.PORTFOLIO.get_number_of_strategy_open_positions_by_symbol(symbol)` |
| Close all longs | `modules.EXECUTION_ENGINE.close_strategy_long_positions_by_symbol(symbol)` |
| Get current bid/ask | `modules.DATA_PROVIDER.get_latest_tick(symbol)` |
| Signal time (safe) | `event.datetime + tf.to_timedelta()` (backtest) or `datetime.now()` (live) |
| Available indicators | `SMA`, `EMA`, `KAMA`, `RSI`, `ATR`, `MACD`, `BollingerBands`, `Stochastic`, `ADX`, `CCI`, `WilliamsR`, `ROC`, `Aroon`, `DonchianChannels`, `KeltnerChannel`, `VWAP`, `ADR`, `Momentum` |
