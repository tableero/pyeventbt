# PyEventBT — Event Flow Diagram

## Main Event Loop

```mermaid
flowchart TD
    START([Strategy.backtest / Strategy.run_live]) --> HOOK_START["HookService.call_callbacks(ON_START)"]
    HOOK_START --> LOOP_START{Queue empty?}

    %% ─── QUEUE EMPTY: FEED NEW BARS ───
    LOOP_START -- "Yes" --> UPDATE_BARS["DataProvider.update_bars()"]
    UPDATE_BARS --> CSV_OR_LIVE{Mode?}
    CSV_OR_LIVE -- "Backtest" --> CSV_GEN["CSVDataConnector\nYields next BarEvent\nfrom generator"]
    CSV_OR_LIVE -- "Live" --> MT5_FETCH["Mt5LiveDataConnector\nmt5.copy_rates_from_pos()"]
    CSV_GEN --> ENQUEUE_BAR["Queue.put(BarEvent)"]
    MT5_FETCH --> ENQUEUE_BAR
    ENQUEUE_BAR --> LOOP_START

    %% ─── QUEUE NOT EMPTY: DEQUEUE ───
    LOOP_START -- "No" --> DEQUEUE["event = Queue.get()"]
    DEQUEUE --> EVENT_TYPE{event.type?}

    %% ════════════════════════════════════════
    %% BAR EVENT
    %% ════════════════════════════════════════
    EVENT_TYPE -- "BAR" --> BAR_HANDLER["TradingDirector._handle_bar_event()"]
    BAR_HANDLER --> PH_BAR["PortfolioHandler.process_bar_event()"]

    PH_BAR --> IS_BASE_TF{Is base\ntimeframe?}
    IS_BASE_TF -- "No" --> SKIP_UPDATE[Skip — no update needed]
    SKIP_UPDATE --> SCHEDULE
    IS_BASE_TF -- "Yes" --> UPDATE_PORTFOLIO["Portfolio._update_portfolio(bar_event)"]

    UPDATE_PORTFOLIO --> EE_UPDATE["ExecutionEngine\n._update_values_and_check_executions_and_fills()"]
    EE_UPDATE --> CHECK_SLTP{"Check SL/TP\nhits on open\npositions"}
    CHECK_SLTP -- "SL/TP hit" --> EMIT_FILL_SLTP["Queue.put(FillEvent)\ndeal=OUT"]
    CHECK_SLTP -- "No hit" --> CHECK_PENDING{"Check pending\norder triggers"}
    EMIT_FILL_SLTP --> CHECK_PENDING
    CHECK_PENDING -- "Pending triggered" --> EXEC_PENDING["Execute pending order\nQueue.put(FillEvent)\ndeal=IN"]
    CHECK_PENDING -- "None triggered" --> FETCH_STATE["Fetch updated:\n• positions\n• pending orders\n• balance / equity"]
    EXEC_PENDING --> FETCH_STATE
    FETCH_STATE --> RECORD_HIST["Record historical\nbalance & equity"]
    RECORD_HIST --> SCHEDULE

    SCHEDULE["ScheduleService.run_scheduled_callbacks()"]
    SCHEDULE --> TF_BOUNDARY{Timeframe\nboundary\ndetected?}
    TF_BOUNDARY -- "Yes" --> EXEC_CALLBACK["Execute @run_every callback\nfn(ScheduledEvent, Modules)"]
    TF_BOUNDARY -- "No" --> SIGNAL_GEN
    EXEC_CALLBACK --> SIGNAL_GEN

    SIGNAL_GEN["SignalEngineService.generate_signal(bar_event)"]
    SIGNAL_GEN --> CALL_ENGINE["Call signal engine:\ncustom fn(bar_event, modules)\nor predefined (MA Crossover)"]
    CALL_ENGINE --> SIG_RESULT{Signal\nreturned?}
    SIG_RESULT -- "None" --> LOOP_START
    SIG_RESULT -- "SignalEvent\nor list" --> ENQUEUE_SIG["Queue.put(SignalEvent)"]
    ENQUEUE_SIG --> LOOP_START

    %% ════════════════════════════════════════
    %% SIGNAL EVENT
    %% ════════════════════════════════════════
    EVENT_TYPE -- "SIGNAL" --> SIG_HANDLER["TradingDirector._handle_signal_event()"]
    SIG_HANDLER --> HOOK_SIGNAL["HookService.call_callbacks(ON_SIGNAL_EVENT)"]
    HOOK_SIGNAL --> PH_SIGNAL["PortfolioHandler.process_signal_event()"]

    PH_SIGNAL --> SIZING["SizingEngineService.get_suggested_order()"]
    SIZING --> SIZING_TYPE{Sizing\nengine?}
    SIZING_TYPE -- "MinSizing" --> MIN_VOL["volume = symbol.volume_min"]
    SIZING_TYPE -- "FixedSizing" --> FIXED_VOL["volume = config.volume"]
    SIZING_TYPE -- "RiskPctSizing" --> RISK_VOL["volume = (equity × risk_pct)\n÷ (SL distance × point_value)"]
    SIZING_TYPE -- "Custom" --> CUSTOM_VOL["volume = custom_fn(signal, modules)"]
    MIN_VOL --> SUGGESTED["SuggestedOrder\n{signal_event, volume}"]
    FIXED_VOL --> SUGGESTED
    RISK_VOL --> SUGGESTED
    CUSTOM_VOL --> SUGGESTED

    SUGGESTED --> RISK["RiskEngineService.assess_order()"]
    RISK --> RISK_TYPE{Risk\nengine?}
    RISK_TYPE -- "Passthrough" --> PASS_VOL["new_volume = original volume"]
    RISK_TYPE -- "Custom" --> CUSTOM_RISK["new_volume = custom_fn(order, modules)"]
    PASS_VOL --> RISK_CHECK{new_volume > 0?}
    CUSTOM_RISK --> RISK_CHECK

    RISK_CHECK -- "Yes" --> CREATE_ORDER["Create OrderEvent\n{symbol, volume, signal_type,\norder_type, price, sl, tp}"]
    RISK_CHECK -- "No (rejected)" --> LOOP_START
    CREATE_ORDER --> ENQUEUE_ORDER["Queue.put(OrderEvent)"]
    ENQUEUE_ORDER --> LOOP_START

    %% ════════════════════════════════════════
    %% ORDER EVENT
    %% ════════════════════════════════════════
    EVENT_TYPE -- "ORDER" --> ORD_HANDLER["TradingDirector._handle_order_event()"]
    ORD_HANDLER --> EE_PROCESS["ExecutionEngine._process_order_event()"]

    EE_PROCESS --> TRADING_ENABLED{Trading\nenabled?}
    TRADING_ENABLED -- "No" --> WARN_DISABLED["Log warning — order skipped"]
    WARN_DISABLED --> LOOP_START
    TRADING_ENABLED -- "Yes" --> ORDER_TYPE{order_type?}

    ORDER_TYPE -- "MARKET" --> MARKET_EXEC["_send_market_order()"]
    ORDER_TYPE -- "LIMIT / STOP" --> PENDING_EXEC["_send_pending_order()\nStored for later trigger"]

    MARKET_EXEC --> EXEC_MODE{Mode?}
    EXEC_MODE -- "Backtest" --> SIM_MARKET["Mt5SimulatorConnector\n• Validate trade values\n• Calculate margin required\n• Check free margin\n• Create TradePosition\n• Generate TradeDeal (IN)"]
    EXEC_MODE -- "Live" --> LIVE_MARKET["Mt5LiveConnector\n• Create TradeRequest\n• mt5.order_send(request)\n• Check retcode"]

    SIM_MARKET --> MARGIN_CHECK{Sufficient\nmargin?}
    MARGIN_CHECK -- "No" --> MARGIN_CALL["Margin call — order rejected"]
    MARGIN_CALL --> LOOP_START
    MARGIN_CHECK -- "Yes" --> EMIT_FILL_IN["Queue.put(FillEvent)\ndeal=IN"]
    LIVE_MARKET --> EMIT_FILL_IN
    PENDING_EXEC --> STORE_PENDING["Store in pending_orders dict\n(triggered on future bar)"]
    STORE_PENDING --> HOOK_ORDER
    EMIT_FILL_IN --> HOOK_ORDER

    HOOK_ORDER["HookService.call_callbacks(ON_ORDER_EVENT)"]
    HOOK_ORDER --> LOOP_START

    %% ════════════════════════════════════════
    %% FILL EVENT
    %% ════════════════════════════════════════
    EVENT_TYPE -- "FILL" --> FILL_HANDLER["TradingDirector._handle_fill_event()"]
    FILL_HANDLER --> PH_FILL["PortfolioHandler.process_fill_event()"]
    PH_FILL --> ARCHIVE["TradeArchiver.archive_trade(fill_event)"]
    ARCHIVE --> LOOP_START

    %% ════════════════════════════════════════
    %% END OF DATA
    %% ════════════════════════════════════════
    LOOP_START -- "Backtest:\nno more data" --> END_CHECK{Open\npositions?}
    END_CHECK -- "Yes" --> CLOSE_ALL["ExecutionEngine\n.close_all_strategy_positions()\n→ emits FillEvent(OUT) per position"]
    CLOSE_ALL --> LOOP_START
    END_CHECK -- "No + queue empty" --> BT_END["PortfolioHandler.process_backtest_end()"]
    BT_END --> FINAL_UPDATE["Portfolio._update_portfolio_end_of_backtest()"]
    FINAL_UPDATE --> EXPORT{Export\nenabled?}
    EXPORT -- "CSV" --> EXPORT_CSV["Export trades CSV + PnL CSV"]
    EXPORT -- "Parquet" --> EXPORT_PQ["Export trades Parquet + PnL Parquet"]
    EXPORT -- "None" --> BUILD_RESULTS
    EXPORT_CSV --> BUILD_RESULTS
    EXPORT_PQ --> BUILD_RESULTS
    BUILD_RESULTS["BacktestResults(pnl, trades)"]
    BUILD_RESULTS --> HOOK_END["HookService.call_callbacks(ON_END)"]
    HOOK_END --> RETURN([Return BacktestResults])

    %% ─── STYLING ───
    style START fill:#4CAF50,color:#fff
    style RETURN fill:#4CAF50,color:#fff
    style BAR_HANDLER fill:#2196F3,color:#fff
    style SIG_HANDLER fill:#FF9800,color:#fff
    style ORD_HANDLER fill:#9C27B0,color:#fff
    style FILL_HANDLER fill:#F44336,color:#fff
    style ENQUEUE_BAR fill:#2196F3,color:#fff
    style ENQUEUE_SIG fill:#FF9800,color:#fff
    style ENQUEUE_ORDER fill:#9C27B0,color:#fff
    style EMIT_FILL_IN fill:#F44336,color:#fff
    style EMIT_FILL_SLTP fill:#F44336,color:#fff
    style HOOK_START fill:#607D8B,color:#fff
    style HOOK_SIGNAL fill:#607D8B,color:#fff
    style HOOK_ORDER fill:#607D8B,color:#fff
    style HOOK_END fill:#607D8B,color:#fff
    style MARGIN_CALL fill:#b71c1c,color:#fff
    style WARN_DISABLED fill:#b71c1c,color:#fff
```

## Live Bar Detection: How the System Knows a Candle Is Complete

In live mode, the system does **not** receive push notifications from MT5 when a new candle closes. Instead, it uses a **polling + timestamp comparison** strategy:

1. **Heartbeat polling** — `TradingDirector._run_live_trading()` runs an infinite loop. Each iteration sleeps for `heartbeat` seconds (configured via `MT5LiveSessionConfig`), then checks for new bars when the event queue is empty.

2. **Only closed bars** — `Mt5LiveDataProvider.get_latest_bar()` calls `mt5.copy_rates_from_pos(symbol, tf, from_pos=1, count=1)`. The `from_pos=1` parameter is the key: position `0` is the **currently forming** (incomplete) candle, position `1` is the most recently **closed** (complete) candle. The system never processes an incomplete bar.

3. **Datetime comparison** — `update_bars()` tracks the last seen bar datetime per symbol and timeframe in `last_bar_tf_datetime[symbol][timeframe]`. On each poll, it compares the returned bar's datetime against the stored value. If the new bar is newer, a `BarEvent` is emitted and the stored datetime is updated. If the datetime is the same, no new candle has closed yet and nothing happens.

```
Live polling timeline (heartbeat = 1s, timeframe = 1min):

 t=0s   poll → bar datetime = 10:01 → same as last seen → no event
 t=1s   poll → bar datetime = 10:01 → same → no event
 t=2s   poll → bar datetime = 10:01 → same → no event
  ...
 t=58s  poll → bar datetime = 10:01 → same → no event
 t=59s  poll → bar datetime = 10:02 → NEW! → emit BarEvent(10:02) → update last seen
 t=60s  poll → bar datetime = 10:02 → same → no event
  ...
```

**Latency implication:** The worst-case delay between a candle closing and the system detecting it is approximately equal to `heartbeat` seconds. A heartbeat of 1 second means the system detects new 1-minute bars within ~1 second of close.

## Simplified Event Pipeline

```mermaid
flowchart LR
    subgraph DataLayer["Data Layer"]
        CSV["CSV Files"]
        MT5_DATA["MT5 Live API"]
    end

    subgraph EventBus["Shared Queue"]
        Q[("queue.Queue")]
    end

    subgraph EventLoop["TradingDirector"]
        direction TB
        DISPATCH["Event Dispatcher"]
    end

    subgraph Handlers["Event Handlers"]
        direction TB
        BAR_H["BAR Handler"]
        SIG_H["SIGNAL Handler"]
        ORD_H["ORDER Handler"]
        FILL_H["FILL Handler"]
    end

    subgraph Engines["Engine Pipeline"]
        direction TB
        SIGNAL_E["Signal Engine\n@custom_signal_engine\nor MACrossover"]
        SIZING_E["Sizing Engine\nMin | Fixed | RiskPct"]
        RISK_E["Risk Engine\nPassthrough | Custom"]
        EXEC_E["Execution Engine\nSimulator | Live MT5"]
    end

    subgraph State["State Management"]
        direction TB
        PORTFOLIO["Portfolio\nbalance, equity\npositions, orders"]
        ARCHIVER["Trade Archiver\nhistorical trades"]
        SCHEDULE["Schedule Service\n@run_every callbacks"]
    end

    CSV --> Q
    MT5_DATA --> Q

    Q --> DISPATCH
    DISPATCH --> BAR_H
    DISPATCH --> SIG_H
    DISPATCH --> ORD_H
    DISPATCH --> FILL_H

    BAR_H -->|"updates"| PORTFOLIO
    BAR_H -->|"triggers"| SCHEDULE
    BAR_H -->|"calls"| SIGNAL_E
    SIGNAL_E -->|"SignalEvent"| Q

    SIG_H -->|"sizes"| SIZING_E
    SIZING_E -->|"SuggestedOrder"| RISK_E
    RISK_E -->|"OrderEvent"| Q

    ORD_H -->|"executes"| EXEC_E
    EXEC_E -->|"FillEvent"| Q

    FILL_H -->|"archives"| ARCHIVER

    style Q fill:#FFC107,color:#000
    style SIGNAL_E fill:#FF9800,color:#fff
    style SIZING_E fill:#FF9800,color:#fff
    style RISK_E fill:#FF9800,color:#fff
    style EXEC_E fill:#9C27B0,color:#fff
    style PORTFOLIO fill:#4CAF50,color:#fff
    style ARCHIVER fill:#4CAF50,color:#fff
    style SCHEDULE fill:#4CAF50,color:#fff
```

## Core Design Pattern (Portable)

The following diagram shows the minimal, reusable pattern that underlies the full system. Any project can adopt this architecture by implementing these three layers:

```mermaid
flowchart TD
    subgraph Pattern["Event-Driven Pattern — 3 Building Blocks"]
        direction TB

        subgraph Block1["1. Typed Events"]
            E1["BarEvent"]
            E2["SignalEvent"]
            E3["OrderEvent"]
            E4["FillEvent"]
        end

        subgraph Block2["2. Shared Queue"]
            Q[("queue.Queue")]
        end

        subgraph Block3["3. Event Loop"]
            LOOP["while running:\n  event = queue.get()\n  handlers[event.type](event)"]
        end
    end

    E1 & E2 & E3 & E4 -->|"produced by components"| Q
    Q -->|"consumed by"| LOOP
    LOOP -->|"dispatches to handler\nthat may produce new events"| Q

    style Q fill:#FFC107,color:#000
    style LOOP fill:#2196F3,color:#fff
    style E1 fill:#2196F3,color:#fff
    style E2 fill:#FF9800,color:#fff
    style E3 fill:#9C27B0,color:#fff
    style E4 fill:#F44336,color:#fff
```

### Integration Points — What to Swap

```mermaid
flowchart LR
    subgraph Swappable["Swap these for your domain"]
        direction TB
        DATA["Data Source\nCSV / API / WebSocket / DB"]
        ALPHA["Signal Logic\nYour strategy / ML model"]
        SIZE["Sizing Logic\nFixed / Risk-based / Custom"]
        RISK["Risk Filters\nPassthrough / Custom rules"]
        EXEC["Execution\nSimulator / Live broker"]
    end

    subgraph Fixed["Keep these unchanged"]
        direction TB
        QUEUE[("Shared Queue")]
        DIRECTOR["Event Loop\n(dispatch by event.type)"]
    end

    DATA -->|"BarEvent"| QUEUE
    QUEUE --> DIRECTOR
    DIRECTOR -->|"BAR"| ALPHA
    ALPHA -->|"SignalEvent"| QUEUE
    DIRECTOR -->|"SIGNAL"| SIZE
    SIZE -->|"SuggestedOrder"| RISK
    RISK -->|"OrderEvent"| QUEUE
    DIRECTOR -->|"ORDER"| EXEC
    EXEC -->|"FillEvent"| QUEUE

    style QUEUE fill:#FFC107,color:#000
    style DIRECTOR fill:#FFC107,color:#000
    style DATA fill:#81C784,color:#000
    style ALPHA fill:#81C784,color:#000
    style SIZE fill:#81C784,color:#000
    style RISK fill:#81C784,color:#000
    style EXEC fill:#81C784,color:#000
```

## Event Lifecycle: Single Bar to Trade

```mermaid
sequenceDiagram
    participant DP as DataProvider
    participant Q as Queue
    participant TD as TradingDirector
    participant PH as PortfolioHandler
    participant SS as ScheduleService
    participant SE as SignalEngine
    participant SZ as SizingEngine
    participant RE as RiskEngine
    participant EE as ExecutionEngine
    participant P as Portfolio
    participant TA as TradeArchiver

    Note over DP,TA: 1. BAR EVENT PHASE
    DP->>Q: put(BarEvent)
    Q->>TD: get() → BarEvent
    TD->>PH: process_bar_event(bar)
    PH->>P: _update_portfolio(bar)
    P->>EE: _update_values_and_check_executions_and_fills(bar)
    EE-->>Q: put(FillEvent) if SL/TP hit
    EE-->>P: updated positions, balance, equity
    P->>P: record historical balance & equity
    TD->>SS: run_scheduled_callbacks(bar)
    SS-->>SS: execute @run_every callbacks if boundary

    Note over DP,TA: 2. SIGNAL EVENT PHASE
    TD->>SE: generate_signal(bar)
    SE->>SE: call user @custom_signal_engine fn
    SE->>Q: put(SignalEvent)
    Q->>TD: get() → SignalEvent
    TD->>PH: process_signal_event(signal)
    PH->>SZ: get_suggested_order(signal)
    SZ-->>PH: SuggestedOrder {volume}
    PH->>RE: assess_order(suggested_order)

    Note over DP,TA: 3. ORDER EVENT PHASE
    RE->>Q: put(OrderEvent) if volume > 0
    Q->>TD: get() → OrderEvent
    TD->>EE: _process_order_event(order)
    EE->>EE: validate + check margin
    EE->>Q: put(FillEvent) deal=IN

    Note over DP,TA: 4. FILL EVENT PHASE
    Q->>TD: get() → FillEvent
    TD->>PH: process_fill_event(fill)
    PH->>TA: archive_trade(fill)
```

## Architecture Limitations: Coupling Points

This diagram highlights the three coupling layers that prevent running components as independent processes.

```mermaid
flowchart TD
    subgraph Coupling1["Coupling Layer 1: SharedData Singleton"]
        SD[("SharedData\n(class-level static attributes)")]
        EE_W["ExecutionEngine\nWRITES: balance, equity,\nmargin, margin_free"]
        SZ_R["SizingEngine\nREADS: account_info.equity"]
        DP_R["DataProvider\nREADS: symbol_info.digits"]
        P_R["Portfolio\nREADS: via ExecutionEngine getters"]

        EE_W -->|"mutates"| SD
        SD -->|"read by"| SZ_R
        SD -->|"read by"| DP_R
        SD -->|"read by"| P_R
    end

    subgraph Coupling2["Coupling Layer 2: Modules Direct References"]
        MOD["Modules object\n{DATA_PROVIDER, EXECUTION_ENGINE, PORTFOLIO}"]
        CB["User Callbacks\n(@signal_engine, @run_every)"]
        CB -->|".get_latest_bars()"| MOD
        CB -->|".get_number_of_positions()"| MOD
        CB -->|".close_short_positions()"| MOD
    end

    subgraph Coupling3["Coupling Layer 3: Synchronous Call Chains"]
        PH2["PortfolioHandler"]
        SZ2["SizingEngine"]
        RE2["RiskEngine"]
        PORT2["Portfolio"]
        EE2["ExecutionEngine"]

        PH2 -->|"sync call\nreturns SuggestedOrder"| SZ2
        PH2 -->|"sync call\nmay queue OrderEvent"| RE2
        PORT2 -->|"sync call\nreturns positions"| EE2
        PORT2 -->|"sync call\nreturns balance"| EE2
    end

    style SD fill:#F44336,color:#fff
    style MOD fill:#FF9800,color:#fff
    style EE_W fill:#EF9A9A,color:#000
    style SZ_R fill:#EF9A9A,color:#000
    style DP_R fill:#EF9A9A,color:#000
    style P_R fill:#EF9A9A,color:#000
    style CB fill:#FFE0B2,color:#000
```

## Distributed Migration: Target Architecture

How the same event flow looks when each component runs as a separate process with a message broker between them.

```mermaid
flowchart LR
    subgraph P1["Process 1: DataProvider"]
        DP3["DataProvider\n(CSV / API / WebSocket)"]
    end

    subgraph Broker["Message Broker (RabbitMQ / Kafka)"]
        T_BAR["topic:\nevent.bar"]
        T_SIG["topic:\nevent.signal"]
        T_SORD["topic:\nevent.suggested_order"]
        T_ORD["topic:\nevent.order"]
        T_FILL["topic:\nevent.fill"]
        T_STATE["topic:\nevent.account_state"]
        T_CMD["topic:\ncommand.*"]
        T_RESP["topic:\nresponse.*"]
    end

    subgraph P2["Process 2: SignalEngine"]
        SE3["SignalEngine\n+ LocalStateCache"]
    end

    subgraph P3["Process 3: SizingEngine"]
        SZ3["SizingEngine\n+ LocalStateCache"]
    end

    subgraph P4["Process 4: RiskEngine"]
        RE3["RiskEngine"]
    end

    subgraph P5["Process 5: ExecutionEngine"]
        EE3["ExecutionEngine\n(Simulator / Live Broker)"]
    end

    subgraph P6["Process 6: Portfolio"]
        PORT3["Portfolio\n+ TradeArchiver"]
    end

    DP3 -->|"BarEvent"| T_BAR
    T_BAR --> SE3
    T_BAR --> PORT3

    SE3 -->|"SignalEvent"| T_SIG
    T_SIG --> SZ3

    SZ3 -->|"SuggestedOrderEvent"| T_SORD
    T_SORD --> RE3

    RE3 -->|"OrderEvent"| T_ORD
    T_ORD --> EE3

    EE3 -->|"FillEvent"| T_FILL
    T_FILL --> PORT3

    EE3 -->|"AccountStateEvent"| T_STATE
    T_STATE --> SE3
    T_STATE --> SZ3
    T_STATE --> PORT3

    SE3 -.->|"ClosePositionsCommand"| T_CMD
    T_CMD -.-> EE3
    EE3 -.->|"ClosePositionsResponse"| T_RESP
    T_RESP -.-> SE3

    style T_BAR fill:#2196F3,color:#fff
    style T_SIG fill:#FF9800,color:#fff
    style T_SORD fill:#FF9800,color:#fff
    style T_ORD fill:#9C27B0,color:#fff
    style T_FILL fill:#F44336,color:#fff
    style T_STATE fill:#4CAF50,color:#fff
    style T_CMD fill:#607D8B,color:#fff
    style T_RESP fill:#607D8B,color:#fff
```

## Recommended Hybrid Architecture

A pragmatic middle ground: distribute only at genuine I/O boundaries, keep tight computation pipelines in one process.

```mermaid
flowchart LR
    subgraph PA["Process A: Market Data"]
        DP4["DataProvider\n(API / WebSocket / CSV)"]
    end

    subgraph MQ["Message Broker"]
        Q1["bars"]
        Q2["orders"]
        Q3["fills + state"]
    end

    subgraph PB["Process B: Strategy (single process)"]
        direction TB
        SE4["SignalEngine"]
        SZ4["SizingEngine"]
        RE4["RiskEngine"]
        SC4["ScheduleService"]
        SE4 -->|"sync"| SZ4
        SZ4 -->|"sync"| RE4
    end

    subgraph PC["Process C: Execution"]
        EE4["ExecutionEngine\n(Simulator / Broker)"]
    end

    subgraph PD["Process D: Monitoring"]
        PORT4["Portfolio\n+ TradeArchiver\n+ Dashboard"]
    end

    DP4 -->|"BarEvent"| Q1
    Q1 --> PB
    PB -->|"OrderEvent"| Q2
    Q2 --> EE4
    EE4 -->|"FillEvent +\nAccountState"| Q3
    Q3 --> PB
    Q3 --> PD

    style PA fill:#81C784,color:#000
    style PB fill:#81C784,color:#000
    style PC fill:#81C784,color:#000
    style PD fill:#81C784,color:#000
    style Q1 fill:#FFC107,color:#000
    style Q2 fill:#FFC107,color:#000
    style Q3 fill:#FFC107,color:#000
```

## Component Contracts & Protocols

Each box is a **contract** (what the component must implement). Each arrow is a **protocol** (what message flows between them and what each side expects).

```mermaid
flowchart TD
    subgraph Contracts["Contract Boundaries"]
        direction TB

        subgraph DP_C["IDataProvider Contract"]
            DP_M1["update_bars() → emits BarEvent"]
            DP_M2["get_latest_bars(symbol, tf, count) → DataFrame"]
            DP_M3["get_latest_tick(symbol) → dict"]
            DP_M4["get_latest_bid/ask(symbol) → Decimal"]
        end

        subgraph SE_C["ISignalEngine Contract"]
            SE_M1["generate_signal(bar, modules)\n→ SignalEvent | list | None"]
        end

        subgraph SZ_C["ISizingEngine Contract"]
            SZ_M1["get_suggested_order(signal, modules)\n→ SuggestedOrder"]
        end

        subgraph RE_C["IRiskEngine Contract"]
            RE_M1["assess_order(suggested_order, modules)\n→ float (volume or 0)"]
        end

        subgraph EE_C["IExecutionEngine Contract"]
            EE_M1["_send_market_order(order) → OrderSendResult"]
            EE_M2["_send_pending_order(order) → OrderSendResult"]
            EE_M3["close_position(ticket) → OrderSendResult"]
            EE_M4["_update_values_and_check_executions_and_fills(bar)"]
            EE_M5["_get_account_balance/equity() → Decimal"]
            EE_M6["_get_strategy_positions() → tuple[OpenPosition]"]
        end

        subgraph PF_C["IPortfolio Contract"]
            PF_M1["_update_portfolio(bar) → None"]
            PF_M2["get_positions(symbol) → tuple[OpenPosition]"]
            PF_M3["get_account_balance/equity() → Decimal"]
        end

        subgraph TA_C["ITradeArchiver Contract"]
            TA_M1["archive_trade(fill) → None"]
            TA_M2["export_csv/parquet/dataframe()"]
        end
    end

    DP_C -->|"BarEvent\n(protocol: chronological,\naligned, no lookahead)"| SE_C
    SE_C -->|"SignalEvent\n(protocol: must set strategy_id,\nsl if RiskPctSizing)"| SZ_C
    SZ_C -->|"SuggestedOrder\n(protocol: volume respects\nsymbol limits)"| RE_C
    RE_C -->|"OrderEvent\n(protocol: only if\nvolume > 0)"| EE_C
    EE_C -->|"FillEvent\n(protocol: must populate\nall cost fields)"| TA_C
    EE_C -->|"FillEvent +\nstate updates"| PF_C

    style DP_C fill:#2196F3,color:#fff
    style SE_C fill:#FF9800,color:#fff
    style SZ_C fill:#FF9800,color:#fff
    style RE_C fill:#FF9800,color:#fff
    style EE_C fill:#9C27B0,color:#fff
    style PF_C fill:#4CAF50,color:#fff
    style TA_C fill:#4CAF50,color:#fff
```

## Multi-Provider & Multi-Broker Patterns

How to extend the architecture with multiple data sources and brokers.

```mermaid
flowchart LR
    subgraph DataProviders["Multiple Data Providers (same IDataProvider contract)"]
        CSV["CSVDataProvider\n(backtest)"]
        MT5D["MT5LiveDataProvider\n(forex live)"]
        BIN["BinanceDataProvider\n(crypto)"]
        YAHOO["YahooDataProvider\n(equities)"]
    end

    COMP["CompositeDataProvider\nroutes by symbol"]

    CSV --> COMP
    MT5D --> COMP
    BIN --> COMP
    YAHOO --> COMP

    COMP -->|"BarEvent"| BUS[("Message Bus")]

    subgraph Strategy["Signal + Sizing + Risk\n(same process)"]
        SE5["SignalEngine"]
        SZ5["SizingEngine"]
        RE5["RiskEngine"]
    end

    BUS --> Strategy

    Strategy -->|"OrderEvent"| BUS

    subgraph Brokers["Multiple Brokers (same IExecutionEngine contract)"]
        MT5E["MT5 Broker\n(forex)"]
        IBKR["IBKR Broker\n(equities)"]
        BINE["Binance Broker\n(crypto)"]
    end

    ROUTER["RoutingExecutionEngine\nroutes by symbol"]

    BUS --> ROUTER
    ROUTER --> MT5E
    ROUTER --> IBKR
    ROUTER --> BINE

    MT5E -->|"FillEvent"| BUS
    IBKR -->|"FillEvent"| BUS
    BINE -->|"FillEvent"| BUS

    subgraph MultiAccount["Multi-Account (same broker, multiple configs)"]
        ACC1["MT5 Account A\n(conservative)"]
        ACC2["MT5 Account B\n(aggressive)"]
    end

    MT5E --> ACC1
    MT5E --> ACC2

    style COMP fill:#2196F3,color:#fff
    style ROUTER fill:#9C27B0,color:#fff
    style BUS fill:#FFC107,color:#000
    style CSV fill:#E3F2FD,color:#000
    style MT5D fill:#E3F2FD,color:#000
    style BIN fill:#E3F2FD,color:#000
    style YAHOO fill:#E3F2FD,color:#000
    style MT5E fill:#E8D5F5,color:#000
    style IBKR fill:#E8D5F5,color:#000
    style BINE fill:#E8D5F5,color:#000
    style ACC1 fill:#F3E5F5,color:#000
    style ACC2 fill:#F3E5F5,color:#000
```
