# Packaging Guide — Python Library + Airflow Integration

> How to structure the trading framework as an installable Python library used in both production (ECS/Lightsail) and research (Airflow parallel backtesting). See also: [Deployment Guide](deployment_guide.md) | [Implementation Guide](implementation_guide.md)

---

## 36.1 The Goal

One Python library, two modes:

```
┌─────────────────────────────────────────────────────────────────┐
│                    trading-engine (pip install)                   │
│                                                                  │
│  Core: Kernel, MessageBus, Cache, Events, Engines               │
│  Adapters: IDataAdapter, IExecutionAdapter (ABCs)               │
│  Built-in: SimulatorAdapter, CSVAdapter                         │
│  Persistence: PersistentCache, RedisEventJournal                │
│  Indicators: SMA, EMA, RSI, ATR, BollingerBands, ...           │
└──────────┬──────────────────────────────────┬───────────────────┘
           │                                   │
    ┌──────▼──────────┐              ┌────────▼────────────────┐
    │  PROD MODE       │              │  RESEARCH MODE           │
    │  (ECS/Lightsail)  │              │  (Airflow/Local)         │
    │                   │              │                          │
    │  PersistentCache  │              │  Cache (in-memory)       │
    │  RedisEventJournal│              │  EventJournal (file)     │
    │  MT5Adapter       │              │  SimulatorAdapter        │
    │  BinanceAdapter   │              │  CSVAdapter              │
    │                   │              │                          │
    │  1 Kernel         │              │  N Kernels in parallel   │
    │  Long-running     │              │  One-shot per backtest   │
    │  State in Redis   │              │  State in memory         │
    └───────────────────┘              └──────────────────────────┘
```

---

## 36.2 Library Structure

```
trading-engine/
├── pyproject.toml                    # Poetry/setuptools config
├── README.md
├── trading_engine/
│   ├── __init__.py                   # Public API exports
│   │
│   ├── core/                         # Framework internals
│   │   ├── kernel.py                 # Kernel (event loop)
│   │   ├── message_bus.py            # MessageBus (pub/sub + req/resp)
│   │   ├── cache.py                  # Cache + PersistentCache
│   │   ├── journal.py                # EventJournal + RedisEventJournal
│   │   ├── snapshot.py               # SnapshotManager
│   │   └── context.py                # StrategyContext (replaces Modules)
│   │
│   ├── events/                       # Event definitions
│   │   ├── types.py                  # EventType enum
│   │   ├── bar.py                    # BarEvent, Bar dataclass
│   │   ├── signal.py                 # SignalEvent, SignalType
│   │   ├── order.py                  # OrderEvent, OrderType
│   │   └── fill.py                   # FillEvent, DealType
│   │
│   ├── engines/                      # Strategy engines
│   │   ├── interfaces.py             # ISignalEngine, ISizingEngine, IRiskEngine
│   │   ├── sizing/                   # Built-in sizing engines
│   │   │   ├── fixed.py
│   │   │   ├── min_volume.py
│   │   │   └── risk_pct.py
│   │   └── risk/                     # Built-in risk engines
│   │       └── passthrough.py
│   │
│   ├── adapters/                     # Data + Execution adapters
│   │   ├── interfaces.py             # IDataAdapter, IExecutionAdapter, AdapterOrderResult
│   │   ├── data/
│   │   │   ├── csv_adapter.py        # CSVDataAdapter (backtest)
│   │   │   ├── mt5_data_adapter.py   # MT5DataAdapter (live)
│   │   │   ├── binance_adapter.py    # BinanceDataAdapter (live)
│   │   │   └── composite.py          # CompositeDataAdapter (multi-source)
│   │   └── execution/
│   │       ├── simulator.py          # SimulatorExecutionAdapter (backtest)
│   │       ├── mt5_adapter.py        # MT5ExecutionAdapter (live)
│   │       ├── ibkr_adapter.py       # IBKRExecutionAdapter (live)
│   │       ├── binance_adapter.py    # BinanceExecutionAdapter (live)
│   │       ├── routing.py            # RoutingExecutionAdapter (multi-broker)
│   │       └── multi_account.py      # MultiAccountExecutionAdapter
│   │
│   ├── indicators/                   # Technical indicators
│   │   ├── sma.py
│   │   ├── ema.py
│   │   ├── rsi.py
│   │   ├── atr.py
│   │   ├── bollinger.py
│   │   └── kama.py
│   │
│   ├── results/                      # Backtest results + plotting
│   │   ├── backtest_results.py
│   │   └── plotting.py
│   │
│   └── utils/
│       ├── decimals.py               # Decimal helpers
│       └── timeframes.py             # Timeframe constants and helpers
│
├── tests/                            # Test suite
│   ├── test_kernel.py
│   ├── test_cache.py
│   ├── test_message_bus.py
│   ├── test_adapters/
│   └── test_engines/
│
└── extras/                           # Optional integrations (not in core package)
    ├── airflow/
    │   ├── dags/
    │   │   ├── backtest_dag.py
    │   │   ├── optimization_dag.py
    │   │   └── walk_forward_dag.py
    │   └── operators/
    │       └── backtest_operator.py
    └── docker/
        ├── Dockerfile
        ├── Dockerfile.airflow
        └── docker-compose.yml
```

---

## 36.3 Public API (`__init__.py`)

```python
"""
trading-engine — Event-driven backtesting and live trading framework.

Usage:
    from trading_engine import Kernel, Cache, MessageBus
    from trading_engine import BarEvent, SignalEvent, OrderEvent, FillEvent
    from trading_engine import StrategyContext, SignalType, OrderType
    from trading_engine import CSVDataAdapter, SimulatorExecutionAdapter
    from trading_engine import indicators
"""

# Core
from trading_engine.core.kernel import Kernel
from trading_engine.core.message_bus import MessageBus
from trading_engine.core.cache import Cache, PersistentCache
from trading_engine.core.journal import EventJournal, RedisEventJournal
from trading_engine.core.context import StrategyContext

# Events
from trading_engine.events.types import EventType
from trading_engine.events.bar import BarEvent, Bar
from trading_engine.events.signal import SignalEvent, SignalType
from trading_engine.events.order import OrderEvent, OrderType
from trading_engine.events.fill import FillEvent, DealType

# Adapter interfaces
from trading_engine.adapters.interfaces import (
    IDataAdapter,
    IExecutionAdapter,
    AdapterOrderResult,
)

# Built-in adapters
from trading_engine.adapters.data.csv_adapter import CSVDataAdapter
from trading_engine.adapters.execution.simulator import SimulatorExecutionAdapter
from trading_engine.adapters.execution.routing import RoutingExecutionAdapter
from trading_engine.adapters.execution.multi_account import MultiAccountExecutionAdapter

# Engine interfaces
from trading_engine.engines.interfaces import (
    ISignalEngine,
    ISizingEngine,
    IRiskEngine,
)

# Built-in engines
from trading_engine.engines.sizing.fixed import FixedSizingEngine
from trading_engine.engines.sizing.min_volume import MinVolumeSizingEngine
from trading_engine.engines.sizing.risk_pct import RiskPctSizingEngine
from trading_engine.engines.risk.passthrough import PassthroughRiskEngine

# Results
from trading_engine.results.backtest_results import BacktestResults

# Indicators
from trading_engine import indicators
```

---

## 36.4 `pyproject.toml`

```toml
[tool.poetry]
name = "trading-engine"
version = "0.1.0"
description = "Event-driven backtesting and live trading framework"
authors = ["Your Name <you@example.com>"]
license = "Apache-2.0"
readme = "README.md"
packages = [{include = "trading_engine"}]

[tool.poetry.dependencies]
python = ">=3.12"
polars = ">=0.20"        # DataFrames for bar data
pydantic = ">=2.0"       # Configuration models

# Optional dependencies for different modes
[tool.poetry.extras]
redis = ["redis"]         # PersistentCache, RedisEventJournal
mt5 = ["MetaTrader5"]     # MT5 adapters (Windows only)
ibkr = ["ib_insync"]      # Interactive Brokers adapter
binance = ["python-binance"]  # Binance adapter
indicators = ["numba"]    # Numba-accelerated indicators
plotting = ["matplotlib", "seaborn"]  # Backtest result plotting
airflow = ["apache-airflow"]  # Airflow integration

[tool.poetry.group.dev.dependencies]
pytest = ">=7.0"
pytest-cov = ">=4.0"

[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"
```

### Installation by mode

```bash
# Research/backtest only (minimal)
pip install trading-engine

# With Redis persistence (for live trading in containers)
pip install trading-engine[redis]

# With MT5 broker (Windows)
pip install trading-engine[mt5,redis]

# With Airflow integration
pip install trading-engine[airflow]

# Everything
pip install trading-engine[redis,mt5,ibkr,binance,indicators,plotting,airflow]
```

---

## 36.5 Usage — Backtest (Local or Airflow Task)

### Simple backtest

```python
from decimal import Decimal
from trading_engine import (
    Kernel, Cache, CSVDataAdapter, SimulatorExecutionAdapter,
    BarEvent, SignalEvent, SignalType, OrderType, StrategyContext,
    FixedSizingEngine, PassthroughRiskEngine, indicators,
)


# ── Define strategy ─────────────────────────────────

def ma_crossover(bar: BarEvent, ctx: StrategyContext):
    bars = ctx.get_latest_bars(bar.symbol, bar.timeframe, 50)
    if bars is None or len(bars) < 50:
        return None

    closes = bars.select("close").to_numpy().flatten()
    fast = indicators.SMA(closes, 10)
    slow = indicators.SMA(closes, 50)

    positions = ctx.get_position_count(symbol=bar.symbol)

    if fast[-1] > slow[-1] and fast[-2] <= slow[-2] and positions == 0:
        return SignalEvent(
            symbol=bar.symbol,
            signal_type=SignalType.BUY,
            order_type=OrderType.MARKET,
            strategy_id="1001",
        )
    if fast[-1] < slow[-1] and fast[-2] >= slow[-2] and positions > 0:
        ctx.close_positions_by_symbol(bar.symbol, direction="BUY")

    return None

ma_crossover.strategy_id = "1001"


# ── Run backtest ─────────────────────────────────────

kernel = Kernel(
    data_adapter=CSVDataAdapter([
        {"symbol": "EURUSD", "timeframe": "1h", "path": "data/eurusd_1h.csv"},
    ]),
    exec_adapter=SimulatorExecutionAdapter(initial_balance=Decimal("10000")),
    signal_engine=ma_crossover,
    sizing_engine=FixedSizingEngine(volume=Decimal("0.1")),
    risk_engine=PassthroughRiskEngine(),
    initial_balance=Decimal("10000"),
)

results = kernel.run()
results.plot()
print(f"Final balance: {results.final_balance}")
print(f"Total trades: {results.total_trades}")
print(f"Sharpe ratio: {results.sharpe_ratio}")
```

### Parameterized backtest (for optimization)

```python
def run_backtest(
    symbol: str,
    fast_period: int,
    slow_period: int,
    risk_pct: float,
    data_path: str,
    initial_balance: Decimal = Decimal("10000"),
) -> dict:
    """
    Self-contained backtest function. Takes parameters, returns results.
    No shared state. Can run in any process/container/Airflow task.
    """

    def signal_engine(bar: BarEvent, ctx: StrategyContext):
        bars = ctx.get_latest_bars(bar.symbol, bar.timeframe, slow_period)
        if bars is None or len(bars) < slow_period:
            return None
        closes = bars.select("close").to_numpy().flatten()
        fast = indicators.SMA(closes, fast_period)
        slow = indicators.SMA(closes, slow_period)
        positions = ctx.get_position_count(symbol=bar.symbol)
        if fast[-1] > slow[-1] and fast[-2] <= slow[-2] and positions == 0:
            return SignalEvent(
                symbol=bar.symbol, signal_type=SignalType.BUY,
                order_type=OrderType.MARKET, strategy_id="1001",
            )
        if fast[-1] < slow[-1] and positions > 0:
            ctx.close_positions_by_symbol(bar.symbol, direction="BUY")
        return None

    signal_engine.strategy_id = "1001"

    kernel = Kernel(
        data_adapter=CSVDataAdapter([
            {"symbol": symbol, "timeframe": "1h", "path": data_path},
        ]),
        exec_adapter=SimulatorExecutionAdapter(initial_balance=initial_balance),
        signal_engine=signal_engine,
        sizing_engine=RiskPctSizingEngine(risk_pct=risk_pct),
        risk_engine=PassthroughRiskEngine(),
        initial_balance=initial_balance,
    )

    results = kernel.run()

    return {
        "fast_period": fast_period,
        "slow_period": slow_period,
        "risk_pct": risk_pct,
        "final_balance": float(results.final_balance),
        "total_trades": results.total_trades,
        "sharpe_ratio": results.sharpe_ratio,
        "max_drawdown": results.max_drawdown,
        "win_rate": results.win_rate,
    }
```

---

## 36.6 Usage — Live Trading (ECS/Lightsail)

```python
import os
from decimal import Decimal
from trading_engine import (
    Kernel, PersistentCache, RedisEventJournal,
    MT5ExecutionAdapter, MT5DataAdapter,
    BarEvent, SignalEvent, SignalType, OrderType, StrategyContext,
    RiskPctSizingEngine, PassthroughRiskEngine,
)


# ── Same strategy function as backtest ───────────────
# (imported from your strategy module, zero code changes)
from my_strategies.ma_crossover import ma_crossover


# ── Configure for live ───────────────────────────────

redis_url = os.environ["REDIS_URL"]
mt5_account = int(os.environ["MT5_ACCOUNT"])
mt5_password = os.environ["MT5_PASSWORD"]
mt5_server = os.environ["MT5_SERVER"]

kernel = Kernel(
    data_adapter=MT5DataAdapter(
        account=mt5_account,
        password=mt5_password,
        server=mt5_server,
        symbols=["EURUSD"],
        timeframes=["1h"],
    ),
    exec_adapter=MT5ExecutionAdapter(
        account=mt5_account,
        password=mt5_password,
        server=mt5_server,
    ),
    signal_engine=ma_crossover,             # SAME function
    sizing_engine=RiskPctSizingEngine(risk_pct=2.0),  # SAME engine
    risk_engine=PassthroughRiskEngine(),     # SAME engine
    initial_balance=Decimal("10000"),

    # Only these two lines change for live:
    cache=PersistentCache(redis_url=redis_url, prefix="eurusd_ma"),
    journal=RedisEventJournal(redis_url=redis_url, stream_key="eurusd_ma:events"),
)

kernel.run()  # Long-running, reconnects on failure
```

**What changed from backtest to live:**
- `CSVDataAdapter` → `MT5DataAdapter`
- `SimulatorExecutionAdapter` → `MT5ExecutionAdapter`
- `Cache()` → `PersistentCache(redis_url=...)`
- `EventJournal("file.jsonl")` → `RedisEventJournal(redis_url=...)`

**What stayed the same:**
- Strategy function (`ma_crossover`)
- Sizing engine
- Risk engine
- Kernel
- MessageBus
- StrategyContext
- All event types

---

## 36.7 Airflow Integration — Parallel Backtesting

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      Airflow (ECS or EC2)                        │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  DAG: optimization_eurusd_ma                              │   │
│  │                                                           │   │
│  │  generate_params ──▶ [backtest_1] ──▶ collect_results    │   │
│  │                      [backtest_2]                         │   │
│  │                      [backtest_3]                         │   │
│  │                      [backtest_4]     ──▶ select_best    │   │
│  │                      ...                                  │   │
│  │                      [backtest_N]     ──▶ deploy (optional)│  │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  Each backtest task:                                             │
│  - Runs in its own process/container (CeleryExecutor or K8s)    │
│  - Imports trading-engine as a library                           │
│  - Creates its own Kernel with in-memory Cache                   │
│  - Completely independent, no shared state                       │
│  - Returns results as a dict (XCom or S3)                        │
└─────────────────────────────────────────────────────────────────┘
```

### Custom Airflow Operator

```python
# extras/airflow/operators/backtest_operator.py

from airflow.models import BaseOperator
from airflow.utils.decorators import apply_defaults


class BacktestOperator(BaseOperator):
    """
    Airflow operator that runs a single backtest using the trading-engine library.

    Accepts strategy parameters, runs the Kernel, returns results via XCom.
    Each task instance is fully independent — no shared state.
    """

    template_fields = ["params"]

    @apply_defaults
    def __init__(
        self,
        strategy_module: str,       # "my_strategies.ma_crossover"
        strategy_function: str,     # "ma_crossover"
        data_configs: list[dict],   # [{"symbol": "EURUSD", "timeframe": "1h", "path": "..."}]
        sizing_config: dict,        # {"type": "risk_pct", "risk_pct": 2.0}
        params: dict = None,        # Strategy-specific parameters (fast_period, slow_period, ...)
        initial_balance: str = "10000",
        **kwargs,
    ):
        super().__init__(**kwargs)
        self.strategy_module = strategy_module
        self.strategy_function = strategy_function
        self.data_configs = data_configs
        self.sizing_config = sizing_config
        self.params = params or {}
        self.initial_balance = initial_balance

    def execute(self, context):
        import importlib
        from decimal import Decimal
        from trading_engine import (
            Kernel, Cache, CSVDataAdapter, SimulatorExecutionAdapter,
            PassthroughRiskEngine,
        )

        # ── Import strategy function ──
        module = importlib.import_module(self.strategy_module)
        strategy_fn = getattr(module, self.strategy_function)

        # ── Apply parameters to strategy ──
        # Strategy function reads params from its closure or a config object
        if hasattr(strategy_fn, "set_params"):
            strategy_fn.set_params(self.params)

        # ── Build sizing engine from config ──
        sizing = self._build_sizing_engine(self.sizing_config)

        # ── Run backtest ──
        kernel = Kernel(
            data_adapter=CSVDataAdapter(self.data_configs),
            exec_adapter=SimulatorExecutionAdapter(
                initial_balance=Decimal(self.initial_balance),
            ),
            signal_engine=strategy_fn,
            sizing_engine=sizing,
            risk_engine=PassthroughRiskEngine(),
            initial_balance=Decimal(self.initial_balance),
        )

        results = kernel.run()

        # ── Return results (stored in XCom) ──
        output = {
            **self.params,
            "final_balance": float(results.final_balance),
            "total_trades": results.total_trades,
            "sharpe_ratio": results.sharpe_ratio,
            "max_drawdown": results.max_drawdown,
            "win_rate": results.win_rate,
            "profit_factor": results.profit_factor,
        }

        self.log.info(f"Backtest complete: {output}")
        return output

    def _build_sizing_engine(self, config: dict):
        from trading_engine import (
            FixedSizingEngine, MinVolumeSizingEngine, RiskPctSizingEngine,
        )
        from decimal import Decimal

        sizing_type = config.get("type", "fixed")
        if sizing_type == "fixed":
            return FixedSizingEngine(volume=Decimal(str(config.get("volume", "0.1"))))
        elif sizing_type == "min":
            return MinVolumeSizingEngine()
        elif sizing_type == "risk_pct":
            return RiskPctSizingEngine(risk_pct=config.get("risk_pct", 2.0))
        else:
            raise ValueError(f"Unknown sizing type: {sizing_type}")
```

### Parameter Optimization DAG

```python
# extras/airflow/dags/optimization_dag.py

from datetime import datetime
from itertools import product
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.utils.task_group import TaskGroup

# Import the custom operator
from operators.backtest_operator import BacktestOperator


# ── Parameter grid ───────────────────────────────────

FAST_PERIODS = [5, 10, 15, 20]
SLOW_PERIODS = [30, 50, 75, 100]
RISK_PCTS = [1.0, 2.0, 3.0]

PARAM_GRID = [
    {"fast_period": f, "slow_period": s, "risk_pct": r}
    for f, s, r in product(FAST_PERIODS, SLOW_PERIODS, RISK_PCTS)
    if f < s  # fast must be shorter than slow
]
# Total combinations: ~36 backtests


# ── DAG definition ───────────────────────────────────

with DAG(
    dag_id="optimization_eurusd_ma_crossover",
    start_date=datetime(2026, 1, 1),
    schedule_interval=None,   # Manual trigger only
    catchup=False,
    max_active_tasks=16,      # Run up to 16 backtests in parallel
    tags=["trading", "optimization"],
) as dag:

    # ── Step 1: Run all backtests in parallel ────────

    with TaskGroup(group_id="backtests") as backtest_group:
        backtest_tasks = []
        for i, params in enumerate(PARAM_GRID):
            task = BacktestOperator(
                task_id=f"bt_f{params['fast_period']}_s{params['slow_period']}_r{int(params['risk_pct']*10)}",
                strategy_module="my_strategies.ma_crossover",
                strategy_function="ma_crossover",
                data_configs=[{
                    "symbol": "EURUSD",
                    "timeframe": "1h",
                    "path": "/data/eurusd_1h.csv",  # Mounted volume or S3
                }],
                sizing_config={"type": "risk_pct", "risk_pct": params["risk_pct"]},
                params=params,
                initial_balance="10000",
            )
            backtest_tasks.append(task)

    # ── Step 2: Collect and rank results ─────────────

    def collect_and_rank(**context):
        import json

        ti = context["ti"]
        results = []

        for task in backtest_tasks:
            result = ti.xcom_pull(task_ids=f"backtests.{task.task_id}")
            if result:
                results.append(result)

        # Rank by Sharpe ratio
        results.sort(key=lambda r: r.get("sharpe_ratio", 0), reverse=True)

        print("=" * 60)
        print("TOP 10 PARAMETER COMBINATIONS")
        print("=" * 60)
        for i, r in enumerate(results[:10]):
            print(f"{i+1}. fast={r['fast_period']}, slow={r['slow_period']}, "
                  f"risk={r['risk_pct']}% → Sharpe={r['sharpe_ratio']:.3f}, "
                  f"DD={r['max_drawdown']:.2%}, WR={r['win_rate']:.1%}")
        print("=" * 60)

        # Save full results
        with open("/results/optimization_results.json", "w") as f:
            json.dump(results, f, indent=2)

        return results[0] if results else None

    collect_task = PythonOperator(
        task_id="collect_and_rank",
        python_callable=collect_and_rank,
    )

    backtest_group >> collect_task
```

### Walk-Forward Optimization DAG

```python
# extras/airflow/dags/walk_forward_dag.py

from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.utils.task_group import TaskGroup
from operators.backtest_operator import BacktestOperator


# ── Walk-forward configuration ───────────────────────

SYMBOL = "EURUSD"
TIMEFRAME = "1h"
TRAIN_MONTHS = 6          # Optimize on 6 months
TEST_MONTHS = 2            # Test on next 2 months
TOTAL_MONTHS = 24          # Total data span
STEP_MONTHS = 2            # Slide window by 2 months

# Generate walk-forward windows
# Each window: train on [start, start+6m), test on [start+6m, start+8m)
windows = []
for offset in range(0, TOTAL_MONTHS - TRAIN_MONTHS - TEST_MONTHS + 1, STEP_MONTHS):
    train_start = datetime(2024, 1, 1) + timedelta(days=30 * offset)
    train_end = train_start + timedelta(days=30 * TRAIN_MONTHS)
    test_end = train_end + timedelta(days=30 * TEST_MONTHS)
    windows.append({
        "window_id": offset // STEP_MONTHS,
        "train_start": train_start.isoformat(),
        "train_end": train_end.isoformat(),
        "test_start": train_end.isoformat(),
        "test_end": test_end.isoformat(),
    })


# ── DAG ──────────────────────────────────────────────

with DAG(
    dag_id="walk_forward_eurusd_ma",
    start_date=datetime(2026, 1, 1),
    schedule_interval=None,
    catchup=False,
    max_active_tasks=32,
    tags=["trading", "walk-forward"],
) as dag:

    for window in windows:
        wid = window["window_id"]

        with TaskGroup(group_id=f"window_{wid}") as window_group:

            # ── Train: optimize parameters on training period ──

            FAST_PERIODS = [5, 10, 15, 20]
            SLOW_PERIODS = [30, 50, 75, 100]

            train_tasks = []
            for f in FAST_PERIODS:
                for s in SLOW_PERIODS:
                    if f >= s:
                        continue
                    task = BacktestOperator(
                        task_id=f"train_f{f}_s{s}",
                        strategy_module="my_strategies.ma_crossover",
                        strategy_function="ma_crossover",
                        data_configs=[{
                            "symbol": SYMBOL,
                            "timeframe": TIMEFRAME,
                            "path": f"/data/{SYMBOL.lower()}_{TIMEFRAME}.csv",
                            "start_date": window["train_start"],
                            "end_date": window["train_end"],
                        }],
                        sizing_config={"type": "risk_pct", "risk_pct": 2.0},
                        params={"fast_period": f, "slow_period": s},
                    )
                    train_tasks.append(task)

            # ── Select best parameters from training ──

            def select_best_params(window_id, **context):
                ti = context["ti"]
                results = []
                for task in train_tasks:
                    result = ti.xcom_pull(
                        task_ids=f"window_{window_id}.{task.task_id}"
                    )
                    if result:
                        results.append(result)
                results.sort(key=lambda r: r.get("sharpe_ratio", 0), reverse=True)
                best = results[0] if results else {"fast_period": 10, "slow_period": 50}
                return best

            select_task = PythonOperator(
                task_id="select_best",
                python_callable=select_best_params,
                op_kwargs={"window_id": wid},
            )

            # ── Test: run best parameters on out-of-sample period ──

            test_task = BacktestOperator(
                task_id="test_oos",
                strategy_module="my_strategies.ma_crossover",
                strategy_function="ma_crossover",
                data_configs=[{
                    "symbol": SYMBOL,
                    "timeframe": TIMEFRAME,
                    "path": f"/data/{SYMBOL.lower()}_{TIMEFRAME}.csv",
                    "start_date": window["test_start"],
                    "end_date": window["test_end"],
                }],
                sizing_config={"type": "risk_pct", "risk_pct": 2.0},
                # params come from select_best via XCom
                params="{{ ti.xcom_pull(task_ids='window_" + str(wid) + ".select_best') }}",
            )

            train_tasks >> select_task >> test_task
```

---

## 36.8 Airflow Executor Options

| Executor | How backtests run | Best for |
|---|---|---|
| **LocalExecutor** | Parallel processes on one machine | Development, small grids (<50 tasks) |
| **CeleryExecutor** | Distributed across Celery workers (ECS tasks or EC2) | Medium grids (50-500 tasks) |
| **KubernetesExecutor** | Each task is a K8s pod | Large grids (500+ tasks), auto-scaling |
| **ECS Executor** (custom) | Each task is an ECS Fargate task | AWS-native, no K8s overhead |

### Celery + ECS setup

```
┌────────────────────┐     ┌────────────────────┐
│ Airflow Webserver   │     │ Airflow Scheduler   │
│ (ECS Service)       │     │ (ECS Service)       │
└────────┬───────────┘     └────────┬───────────┘
         │                          │
         └──────────┬───────────────┘
                    │
            ┌───────▼───────┐
            │  Redis/SQS     │  ← Celery broker
            │  (ElastiCache)  │
            └───────┬───────┘
                    │
    ┌───────────────┼───────────────┐
    │               │               │
┌───▼───┐     ┌───▼───┐     ┌───▼───┐
│Worker 1│     │Worker 2│     │Worker N│   ← ECS Tasks (auto-scale)
│        │     │        │     │        │
│ import │     │ import │     │ import │
│trading │     │trading │     │trading │
│-engine │     │-engine │     │-engine │
└───┬────┘     └───┬────┘     └───┬────┘
    │              │              │
    └──────────────┼──────────────┘
                   │
           ┌───────▼───────┐
           │  S3             │
           │  - CSV data     │  ← Shared data, results
           │  - Results JSON │
           └────────────────┘
```

Each worker is an ECS Fargate task running `pip install trading-engine`. It pulls data from S3, runs the backtest in memory, pushes results back. No Redis needed for backtests — each task is fully independent.

---

## 36.9 Data Management for Airflow

### Problem

Backtest tasks need access to CSV data. Options:

| Approach | How | Latency | Cost |
|---|---|---|---|
| **S3 + download at task start** | Worker downloads CSV from S3 before backtest | Seconds (first task), cached after | Low |
| **EFS shared volume** | All workers mount same EFS filesystem | Immediate | Medium |
| **Baked into Docker image** | CSV files included in worker image | Immediate | High (large images) |
| **Polars read from S3 directly** | `pl.read_csv("s3://bucket/data.csv")` | Streaming | Low |

### Recommended: S3 + Polars direct read

```python
# In CSVDataAdapter, support S3 paths natively
class CSVDataAdapter(IDataAdapter):
    def connect(self) -> bool:
        import polars as pl
        for cfg in self._configs:
            path = cfg["path"]
            if path.startswith("s3://"):
                # Polars reads S3 natively with storage_options
                self._dataframes[key] = pl.read_csv(
                    path,
                    storage_options={"aws_region": "eu-west-1"},
                )
            else:
                self._dataframes[key] = pl.read_csv(path)
        return True
```

Then in Airflow DAGs, just point to S3:

```python
data_configs=[{
    "symbol": "EURUSD",
    "timeframe": "1h",
    "path": "s3://my-trading-data/eurusd_1h.csv",
}]
```

---

## 36.10 Summary — Same Library, Two Modes

```
┌──────────────────────────────────────────────────────────┐
│                   trading-engine (PyPI)                    │
│                                                           │
│  pip install trading-engine           # backtest          │
│  pip install trading-engine[redis]    # live trading      │
│  pip install trading-engine[airflow]  # parallel research │
└──────────┬───────────────────────────────┬───────────────┘
           │                                │
  ┌────────▼────────────┐       ┌──────────▼──────────────┐
  │  RESEARCH MODE       │       │  PRODUCTION MODE         │
  │                      │       │                          │
  │  Airflow DAG:        │       │  ECS Task:               │
  │  ┌────┐ ┌────┐      │       │  ┌──────────────────┐   │
  │  │BT 1│ │BT 2│ ...  │       │  │ Kernel            │   │
  │  └────┘ └────┘      │       │  │ PersistentCache   │   │
  │  Each task:          │       │  │ RedisEventJournal │   │
  │  - In-memory Cache   │       │  │ MT5Adapter        │   │
  │  - CSVDataAdapter    │       │  │                   │   │
  │  - SimulatorAdapter  │       │  │ Long-running      │   │
  │  - Independent       │       │  │ State in Redis    │   │
  │  - One-shot          │       │  │ Recoverable       │   │
  │                      │       │  └──────────────────┘   │
  │  Results → S3/XCom   │       │                          │
  │                      │       │  Monitoring → CloudWatch │
  │  Optimization:       │       │  Alerts → SNS/Slack      │
  │  - Parameter grids   │       │                          │
  │  - Walk-forward      │       │  Scales to N strategies  │
  │  - Monte Carlo       │       │  via N ECS tasks         │
  └──────────────────────┘       └──────────────────────────┘
```

**The library doesn't know or care which mode it's running in.** The caller (Airflow task or ECS entrypoint) chooses the adapters and cache backend. The Kernel, MessageBus, engines, and strategy functions are identical.

---
