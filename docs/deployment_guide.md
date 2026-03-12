# Deployment Guide — Containerized Trading with Externalized State

> How to deploy the target architecture on ECS, Lightsail, or any container platform. See also: [Implementation Guide](implementation_guide.md) | [CBP Diagrams](cbp_diagrams.md) | [Architecture Comparison](architecture_comparison.md)

---

## 35.1 The Problem: In-Memory State Doesn't Survive Restarts

Every framework except NautilusTrader keeps state in-memory:

| Framework | State Storage | Survives container restart? |
|---|---|---|
| NautilusTrader | Redis + PostgreSQL | Yes |
| QuantConnect LEAN | Local JSON files | No (lost with container) |
| Zipline | In-memory + local files | No |
| Backtrader | In-memory only | No |
| VectorBT | In-memory only | No |
| PyEventBT | `SharedData` singleton | No |

For a trading system running in ECS or Lightsail, this is not acceptable. If the container restarts (deploy, crash, scaling event), you lose:
- Open position tracking
- Account balance/equity state
- Pending order state
- Event journal (audit trail)

The solution: **externalize all state to Redis + a durable store (PostgreSQL/S3)**.

---

## 35.2 Storage Architecture — Three Stores, Each With a Clear Purpose

No PostgreSQL in production. Three stores:

```
Strategy A (ECS) ──▶ Redis (ElastiCache)     ◄── hot state, per-strategy prefix
Strategy B (ECS) ──▶   ├── strategy_a:*
Strategy C (ECS) ──▶   ├── strategy_b:*
                       └── strategy_c:*

All strategies ──────▶ DynamoDB              ◄── shared coordination
                       ├── strategies table   (which strategies exist, status, config)
                       ├── exposure table     (total exposure across all strategies)
                       └── locks table        (e.g., "freeze all trading" flag)

All strategies ──────▶ S3                    ◄── cold storage + reporting
                       ├── trades/            (Parquet, partitioned by date/strategy)
                       ├── events/            (archived journal entries)
                       ├── backtests/         (results from Airflow)
                       └── data/              (historical bars CSV)
                              │
                              ▼
                       Athena ◄── cross-strategy reporting, ad-hoc SQL
```

### Hot state → Redis (per-strategy, fast, ephemeral)

| Data | Why Redis | Update frequency |
|---|---|---|
| Account state (balance, equity, margin) | Sub-ms reads for sizing/risk decisions | Every fill |
| Open positions | Must know what's open to avoid duplicates | Every fill |
| Pending orders | Must track for SL/TP and cancel operations | Every order/fill |
| Event journal (Redis Streams) | Append-only log for crash recovery | Every event |
| Symbol info (digits, volume_min, etc.) | Needed for sizing calculations | On connect |
| Last processed event sequence | Resume point after restart | Every event |

Each strategy uses a different Redis prefix (`strategy_a:*`, `strategy_b:*`). Fully isolated. If Redis data is lost, the Kernel reconciles with the broker on next startup.

### Shared state → DynamoDB (cross-strategy coordination)

| Table | Keys | What it holds | Why DynamoDB |
|---|---|---|---|
| `strategies` | PK: `strategy_id` | Status (running/stopped), config, last heartbeat, deploy timestamp | Shared between N containers, durable, serverless |
| `exposure` | PK: `strategy_id`, SK: `symbol` | Current position size per strategy per symbol | Post-trade risk service reads all rows to compute total exposure |
| `locks` | PK: `lock_name` | Global flags: "freeze_trading", "market_closed", "max_drawdown_hit" | Any strategy reads, monitoring service writes |
| `alerts` | PK: `alert_id`, SK: `timestamp` | Alert history (drawdown breached, connection lost, etc.) | Append-only, queryable by time |

DynamoDB makes sense here because:
- Simple key-value / key-sort access patterns (no joins needed)
- Shared between N strategy containers that don't share Redis
- Serverless — no instance to manage, scales automatically
- Global Tables if you ever go multi-region
- Single-digit millisecond reads are fine for coordination (not in the hot path)

### Cold storage → S3/Parquet (reporting, history, compliance)

| Path | Format | What | Queried by |
|---|---|---|---|
| `s3://bucket/trades/strategy_id=X/date=2026-03-12/` | Parquet | Completed trade records (fills) | Athena for cross-strategy reporting |
| `s3://bucket/events/strategy_id=X/date=2026-03-12/` | Parquet | Archived event journal entries | Athena for replay/audit |
| `s3://bucket/backtests/run_id=abc123/` | Parquet + JSON | Backtest results, equity curves, parameters | Airflow collect step |
| `s3://bucket/data/eurusd_1h.csv` | CSV | Historical bar data for backtesting | Airflow backtest tasks, CSVDataAdapter |
| `s3://bucket/snapshots/strategy_id=X/` | JSON | Cache snapshots for recovery | Kernel on startup (fallback if Redis empty) |

Cross-strategy reporting is Athena queries on S3 Parquet — no database needed:

```sql
-- Total PnL across all strategies this month
SELECT strategy_id, SUM(gross_profit) as pnl
FROM trades
WHERE date >= '2026-03-01'
GROUP BY strategy_id
ORDER BY pnl DESC;

-- Exposure per symbol across all strategies
SELECT symbol, SUM(volume) as total_exposure
FROM trades
WHERE deal = 'IN'
  AND position_id NOT IN (SELECT position_id FROM trades WHERE deal = 'OUT')
GROUP BY symbol;
```

### Why NOT PostgreSQL in prod

| Concern | Answer |
|---|---|
| Complex queries on trades? | Athena on S3 Parquet — cheaper, serverless, no instance to manage |
| Cross-strategy coordination? | DynamoDB — serverless, simpler than RDS for key-value patterns |
| Compliance audit trail? | S3 Parquet — immutable, versioned, cheaper than RDS storage |
| Joins between tables? | Not needed. Trades are self-contained. Cross-strategy = Athena. |

PostgreSQL is only needed locally because **Airflow requires it** as its metadata database. The trading engine itself never touches PostgreSQL.

### Local / Dev vs Prod

```
LOCAL / DEV                          PROD (AWS)
─────────────                        ──────────

PostgreSQL ◄── Airflow metadata      (not needed)
Redis      ◄── strategy hot state    Redis (ElastiCache) ◄── hot state
                                     DynamoDB ◄── shared coordination
                                     S3 + Athena ◄── cold storage + reporting
```

---

## 35.3 Redis-Backed Cache

### Contract

Same `ICache` read/write interface as the in-memory Cache (section 33.1 of the Implementation Guide). The Redis-backed version is a drop-in replacement.

### Behavior

| Rule | Description |
|---|---|
| **Single writer** | Same as in-memory: only the Kernel writes |
| **Write-through** | Every write to Cache also writes to Redis. Read from local memory first, fall back to Redis. |
| **Startup rehydration** | On container start, load all state from Redis into local memory |
| **TTL on hot state** | Positions and account state have no TTL (persist until explicitly removed). Symbol info can have a long TTL (e.g., 24h). |
| **Atomic updates** | Use Redis transactions (MULTI/EXEC) when updating multiple keys that must be consistent (e.g., account state after a fill) |

### Reference Implementation

```python
import json
import redis
from decimal import Decimal
from datetime import datetime


class RedisCacheBackend:
    """
    Redis persistence layer for Cache.
    Wraps a Redis client and handles serialization.
    """

    def __init__(self, redis_url: str = "redis://localhost:6379", prefix: str = "trading"):
        self._redis = redis.from_url(redis_url, decode_responses=True)
        self._prefix = prefix

    def _key(self, *parts) -> str:
        return f"{self._prefix}:{':'.join(str(p) for p in parts)}"

    # ── Account State ────────────────────────────────

    def save_account(self, account: AccountState):
        self._redis.hset(self._key("account"), mapping={
            "balance": str(account.balance),
            "equity": str(account.equity),
            "margin": str(account.margin),
            "margin_free": str(account.margin_free),
            "currency": account.currency,
        })

    def load_account(self) -> AccountState | None:
        data = self._redis.hgetall(self._key("account"))
        if not data:
            return None
        return AccountState(
            balance=Decimal(data["balance"]),
            equity=Decimal(data["equity"]),
            margin=Decimal(data["margin"]),
            margin_free=Decimal(data["margin_free"]),
            currency=data["currency"],
        )

    # ── Positions ────────────────────────────────────

    def save_position(self, pos: PositionSnapshot):
        self._redis.hset(self._key("positions", pos.ticket), mapping={
            "ticket": pos.ticket,
            "symbol": pos.symbol,
            "direction": pos.direction,
            "volume": str(pos.volume),
            "price_entry": str(pos.price_entry),
            "unrealized_pnl": str(pos.unrealized_pnl),
            "strategy_id": pos.strategy_id,
            "sl": str(pos.sl) if pos.sl else "",
            "tp": str(pos.tp) if pos.tp else "",
        })
        # Add to position set for enumeration
        self._redis.sadd(self._key("position_tickets"), pos.ticket)

    def remove_position(self, ticket: int):
        self._redis.delete(self._key("positions", ticket))
        self._redis.srem(self._key("position_tickets"), ticket)

    def load_all_positions(self) -> list[PositionSnapshot]:
        tickets = self._redis.smembers(self._key("position_tickets"))
        positions = []
        for ticket in tickets:
            data = self._redis.hgetall(self._key("positions", ticket))
            if data:
                positions.append(PositionSnapshot(
                    ticket=int(data["ticket"]),
                    symbol=data["symbol"],
                    direction=data["direction"],
                    volume=Decimal(data["volume"]),
                    price_entry=Decimal(data["price_entry"]),
                    unrealized_pnl=Decimal(data["unrealized_pnl"]),
                    strategy_id=data["strategy_id"],
                    sl=Decimal(data["sl"]) if data["sl"] else None,
                    tp=Decimal(data["tp"]) if data["tp"] else None,
                ))
        return positions

    # ── Pending Orders ───────────────────────────────

    def save_pending_order(self, order: PendingOrderSnapshot):
        self._redis.hset(self._key("pending_orders", order.ticket), mapping={
            "ticket": order.ticket,
            "symbol": order.symbol,
            "order_type": order.order_type,
            "volume": str(order.volume),
            "price": str(order.price),
            "strategy_id": order.strategy_id,
            "sl": str(order.sl) if order.sl else "",
            "tp": str(order.tp) if order.tp else "",
        })
        self._redis.sadd(self._key("pending_order_tickets"), order.ticket)

    def remove_pending_order(self, ticket: int):
        self._redis.delete(self._key("pending_orders", ticket))
        self._redis.srem(self._key("pending_order_tickets"), ticket)

    def load_all_pending_orders(self) -> list[PendingOrderSnapshot]:
        tickets = self._redis.smembers(self._key("pending_order_tickets"))
        orders = []
        for ticket in tickets:
            data = self._redis.hgetall(self._key("pending_orders", ticket))
            if data:
                orders.append(PendingOrderSnapshot(
                    ticket=int(data["ticket"]),
                    symbol=data["symbol"],
                    order_type=data["order_type"],
                    volume=Decimal(data["volume"]),
                    price=Decimal(data["price"]),
                    strategy_id=data["strategy_id"],
                    sl=Decimal(data["sl"]) if data["sl"] else None,
                    tp=Decimal(data["tp"]) if data["tp"] else None,
                ))
        return orders

    # ── Symbol Info ──────────────────────────────────

    def save_symbol_info(self, symbol: str, info: SymbolState):
        self._redis.hset(self._key("symbols", symbol), mapping={
            "digits": info.digits,
            "volume_min": str(info.volume_min),
            "volume_max": str(info.volume_max),
            "volume_step": str(info.volume_step),
            "point": str(info.point),
            "trade_contract_size": str(info.trade_contract_size),
        })

    def load_symbol_info(self, symbol: str) -> SymbolState | None:
        data = self._redis.hgetall(self._key("symbols", symbol))
        if not data:
            return None
        return SymbolState(
            digits=int(data["digits"]),
            volume_min=Decimal(data["volume_min"]),
            volume_max=Decimal(data["volume_max"]),
            volume_step=Decimal(data["volume_step"]),
            point=Decimal(data["point"]),
            trade_contract_size=Decimal(data["trade_contract_size"]),
        )

    # ── Sequence tracking ────────────────────────────

    def save_last_sequence(self, seq: int):
        self._redis.set(self._key("last_sequence"), seq)

    def load_last_sequence(self) -> int:
        val = self._redis.get(self._key("last_sequence"))
        return int(val) if val else 0

    # ── Full flush ───────────────────────────────────

    def flush_all(self):
        """Remove all keys with this prefix. Use for test cleanup."""
        keys = self._redis.keys(f"{self._prefix}:*")
        if keys:
            self._redis.delete(*keys)
```

### Write-Through Cache (wraps in-memory + Redis)

```python
class PersistentCache(Cache):
    """
    Drop-in replacement for Cache. Same read/write interface.
    Writes go to both local memory AND Redis.
    Reads come from local memory (fast path).
    On startup, rehydrates from Redis.
    """

    def __init__(self, redis_url: str = "redis://localhost:6379", prefix: str = "trading"):
        super().__init__()
        self._backend = RedisCacheBackend(redis_url=redis_url, prefix=prefix)

    def rehydrate(self):
        """Load all state from Redis into local memory. Call on startup."""
        account = self._backend.load_account()
        if account:
            self._account = account

        for pos in self._backend.load_all_positions():
            self._positions[pos.ticket] = pos

        for order in self._backend.load_all_pending_orders():
            self._pending_orders[order.ticket] = order

        # Symbol info would be loaded per-symbol as needed or bulk-loaded

    # ── Override write methods to persist to Redis ───

    def update_account(self, balance, equity, margin, margin_free):
        super().update_account(balance, equity, margin, margin_free)
        self._backend.save_account(self._account)

    def update_position(self, snapshot: PositionSnapshot):
        super().update_position(snapshot)
        self._backend.save_position(snapshot)

    def remove_position(self, ticket: int):
        super().remove_position(ticket)
        self._backend.remove_position(ticket)

    def set_symbol_info(self, symbol: str, info: SymbolState):
        super().set_symbol_info(symbol, info)
        self._backend.save_symbol_info(symbol, info)

    def update_pending_order(self, snapshot: PendingOrderSnapshot):
        super().update_pending_order(snapshot)
        self._backend.save_pending_order(snapshot)

    def remove_pending_order(self, ticket: int):
        super().remove_pending_order(ticket)
        self._backend.remove_pending_order(ticket)

    def clear_all(self):
        super().clear_all()
        self._backend.flush_all()
```

### Contract / Behavior / Protocol

#### Contract

Same as `ICache` + `ICacheWriter` from section 33.1. No new methods. `PersistentCache` is a transparent drop-in.

#### Behavior

| Rule | Description |
|---|---|
| **Write-through** | Every local write also writes to Redis. No write is considered complete until Redis confirms. |
| **Read-local** | All reads come from local memory. Redis is not hit on reads (performance). |
| **Rehydrate on startup** | `rehydrate()` must be called before `run()`. Loads all Redis state into local memory. |
| **Atomic account updates** | Account state (balance + equity + margin + margin_free) written as a single Redis HSET. |
| **Prefix isolation** | Multiple strategy instances can share one Redis by using different prefixes. |

#### Protocol

```
Normal operation (same as in-memory, plus Redis writes):

    Kernel._on_fill()
        │
        ├── cache.update_position(snapshot)
        │       ├── local memory update
        │       └── Redis HSET trading:positions:{ticket}
        │
        └── cache.update_account(balance, equity, ...)
                ├── local memory update
                └── Redis HSET trading:account

Container restart / recovery:

    Kernel.__init__()
        │
        └── cache.rehydrate()
                ├── Redis HGETALL trading:account → local AccountState
                ├── Redis SMEMBERS trading:position_tickets → for each:
                │       └── Redis HGETALL trading:positions:{ticket} → local PositionSnapshot
                └── Redis SMEMBERS trading:pending_order_tickets → for each:
                        └── Redis HGETALL trading:pending_orders:{ticket} → local PendingOrderSnapshot
```

---

## 35.4 Redis-Backed Event Journal

For containerized deployment, the file-based `EventJournal` is not sufficient — the file is lost when the container restarts. Two options:

### Option A: Redis Streams (recommended for live trading)

Redis Streams are an append-only log data structure — a perfect match for event journaling.

```python
class RedisEventJournal:
    """
    Event journal backed by Redis Streams.
    Append-only, ordered, with built-in sequence IDs.
    """

    def __init__(self, redis_url: str = "redis://localhost:6379",
                 stream_key: str = "trading:events"):
        self._redis = redis.from_url(redis_url, decode_responses=True)
        self._stream_key = stream_key

    def record(self, topic: str, event) -> str:
        """
        Append event to Redis Stream. Returns stream ID (e.g., "1678234567890-0").
        Redis Streams auto-generate monotonic IDs.
        """
        data = {
            "topic": topic,
            "type": event.__class__.__name__,
            "data": json.dumps(self._serialize_event(event), default=str),
            "ts": datetime.now().isoformat(),
        }
        return self._redis.xadd(self._stream_key, data)

    def replay(self, from_id: str = "0-0"):
        """Yield events from a stream ID onwards."""
        entries = self._redis.xrange(self._stream_key, min=from_id)
        for stream_id, data in entries:
            yield {
                "id": stream_id,
                "topic": data["topic"],
                "type": data["type"],
                "data": json.loads(data["data"]),
                "ts": data["ts"],
            }

    def get_last_id(self) -> str | None:
        """Get the ID of the last entry in the stream."""
        info = self._redis.xinfo_stream(self._stream_key)
        return info.get("last-generated-id")

    def trim(self, max_length: int = 100000):
        """Keep only the last N entries. Call periodically to bound memory."""
        self._redis.xtrim(self._stream_key, maxlen=max_length)

    @staticmethod
    def _serialize_event(event) -> dict:
        # Same serialization as file-based journal
        result = {}
        for key, value in event.__dict__.items():
            if isinstance(value, Decimal):
                result[key] = str(value)
            elif isinstance(value, datetime):
                result[key] = value.isoformat()
            elif hasattr(value, "value") and hasattr(value, "name"):
                result[key] = value.value
            else:
                result[key] = value
        return result

    def close(self):
        pass  # Redis connection managed by pool
```

### Option B: S3 for durable long-term journal

Redis Streams can be trimmed (bounded memory). For a permanent audit trail, periodically flush to S3 as Parquet:

```python
class DurableEventJournal:
    """
    Writes to Redis Stream (fast, for recovery) AND S3 (durable, for audit).
    Periodic flush: batch events from Redis Stream → S3 Parquet.
    """

    def __init__(self, redis_url: str, s3_bucket: str, s3_prefix: str = "events"):
        self._redis_journal = RedisEventJournal(redis_url)
        self._s3_bucket = s3_bucket
        self._s3_prefix = s3_prefix
        self._flush_buffer = []
        self._flush_interval = 1000  # flush every N events

    def record(self, topic: str, event) -> str:
        stream_id = self._redis_journal.record(topic, event)

        self._flush_buffer.append({
            "stream_id": stream_id,
            "topic": topic,
            "type": event.__class__.__name__,
            "data": json.dumps(
                self._redis_journal._serialize_event(event), default=str),
            "ts": datetime.now().isoformat(),
        })

        if len(self._flush_buffer) >= self._flush_interval:
            self._flush_to_s3()

        return stream_id

    def _flush_to_s3(self):
        if not self._flush_buffer:
            return
        import polars as pl
        import boto3
        from io import BytesIO

        df = pl.DataFrame(self._flush_buffer)
        buffer = BytesIO()
        df.write_parquet(buffer)
        buffer.seek(0)

        date = datetime.now().strftime("%Y-%m-%d")
        ts = datetime.now().strftime("%H%M%S")
        key = f"{self._s3_prefix}/date={date}/{ts}.parquet"

        s3 = boto3.client("s3")
        s3.upload_fileobj(buffer, self._s3_bucket, key)
        self._flush_buffer.clear()

    def replay(self, from_id: str = "0-0"):
        # Recovery: use Redis (fast)
        return self._redis_journal.replay(from_id)

    def close(self):
        self._flush_to_s3()  # flush remaining events
```

---

## 35.4b DynamoDB — Shared Coordination Across Strategies

### What it holds

DynamoDB stores state that must be visible across all strategy containers. This is NOT in the hot trading path — it's read/written on startup, on fills, and by monitoring services.

### Reference Implementation

```python
import boto3
from datetime import datetime
from decimal import Decimal


class StrategyCoordinator:
    """
    DynamoDB-backed shared coordination layer.
    Used for cross-strategy visibility: which strategies are running,
    total exposure per symbol, global trading locks.
    """

    def __init__(self, table_prefix: str = "trading", region: str = "eu-west-1"):
        self._dynamo = boto3.resource("dynamodb", region_name=region)
        self._strategies_table = self._dynamo.Table(f"{table_prefix}_strategies")
        self._exposure_table = self._dynamo.Table(f"{table_prefix}_exposure")
        self._locks_table = self._dynamo.Table(f"{table_prefix}_locks")

    # ── Strategy Registry ────────────────────────────

    def register_strategy(self, strategy_id: str, config: dict):
        """Called on Kernel startup. Marks strategy as running."""
        self._strategies_table.put_item(Item={
            "strategy_id": strategy_id,
            "status": "running",
            "config": config,
            "started_at": datetime.now().isoformat(),
            "last_heartbeat": datetime.now().isoformat(),
        })

    def heartbeat(self, strategy_id: str):
        """Called periodically (e.g., every 60s) to signal liveness."""
        self._strategies_table.update_item(
            Key={"strategy_id": strategy_id},
            UpdateExpression="SET last_heartbeat = :ts",
            ExpressionAttributeValues={":ts": datetime.now().isoformat()},
        )

    def deregister_strategy(self, strategy_id: str):
        """Called on Kernel shutdown."""
        self._strategies_table.update_item(
            Key={"strategy_id": strategy_id},
            UpdateExpression="SET #s = :status, stopped_at = :ts",
            ExpressionAttributeNames={"#s": "status"},
            ExpressionAttributeValues={
                ":status": "stopped",
                ":ts": datetime.now().isoformat(),
            },
        )

    def get_all_strategies(self) -> list[dict]:
        """List all registered strategies and their status."""
        response = self._strategies_table.scan()
        return response.get("Items", [])

    # ── Exposure Tracking ────────────────────────────

    def update_exposure(self, strategy_id: str, symbol: str, volume: Decimal):
        """Called after every fill. Updates this strategy's exposure for a symbol."""
        self._exposure_table.put_item(Item={
            "strategy_id": strategy_id,
            "symbol": symbol,
            "volume": volume,
            "updated_at": datetime.now().isoformat(),
        })

    def get_total_exposure(self, symbol: str) -> Decimal:
        """
        Get total exposure across ALL strategies for a symbol.
        Used by post-trade risk monitoring.
        """
        response = self._exposure_table.query(
            IndexName="symbol-index",
            KeyConditionExpression="symbol = :sym",
            ExpressionAttributeValues={":sym": symbol},
        )
        return sum(Decimal(str(item["volume"])) for item in response.get("Items", []))

    def get_portfolio_exposure(self) -> dict[str, Decimal]:
        """Get total exposure per symbol across all strategies."""
        response = self._exposure_table.scan()
        totals = {}
        for item in response.get("Items", []):
            sym = item["symbol"]
            vol = Decimal(str(item["volume"]))
            totals[sym] = totals.get(sym, Decimal("0")) + vol
        return totals

    # ── Global Locks ─────────────────────────────────

    def set_lock(self, lock_name: str, value: str, reason: str = ""):
        """
        Set a global lock. All strategies should check locks before trading.
        Examples: "freeze_trading", "max_drawdown_hit", "market_closed"
        """
        self._locks_table.put_item(Item={
            "lock_name": lock_name,
            "value": value,
            "reason": reason,
            "set_at": datetime.now().isoformat(),
        })

    def get_lock(self, lock_name: str) -> str | None:
        """Check a global lock value."""
        response = self._locks_table.get_item(Key={"lock_name": lock_name})
        item = response.get("Item")
        return item.get("value") if item else None

    def is_trading_frozen(self) -> bool:
        """Check if trading is globally frozen."""
        return self.get_lock("freeze_trading") == "true"


# ── DynamoDB Table Definitions (CloudFormation/CDK) ──

DYNAMO_TABLES = {
    "trading_strategies": {
        "KeySchema": [{"AttributeName": "strategy_id", "KeyType": "HASH"}],
        "AttributeDefinitions": [
            {"AttributeName": "strategy_id", "AttributeType": "S"},
        ],
        "BillingMode": "PAY_PER_REQUEST",
    },
    "trading_exposure": {
        "KeySchema": [
            {"AttributeName": "strategy_id", "KeyType": "HASH"},
            {"AttributeName": "symbol", "KeyType": "RANGE"},
        ],
        "AttributeDefinitions": [
            {"AttributeName": "strategy_id", "AttributeType": "S"},
            {"AttributeName": "symbol", "AttributeType": "S"},
        ],
        "GlobalSecondaryIndexes": [{
            "IndexName": "symbol-index",
            "KeySchema": [{"AttributeName": "symbol", "KeyType": "HASH"}],
            "Projection": {"ProjectionType": "ALL"},
        }],
        "BillingMode": "PAY_PER_REQUEST",
    },
    "trading_locks": {
        "KeySchema": [{"AttributeName": "lock_name", "KeyType": "HASH"}],
        "AttributeDefinitions": [
            {"AttributeName": "lock_name", "AttributeType": "S"},
        ],
        "BillingMode": "PAY_PER_REQUEST",
    },
}
```

### How the Kernel uses it

```python
class Kernel:
    def __init__(self, config: dict):
        # ...existing init...

        # DynamoDB coordination (optional, for multi-strategy setups)
        if config.get("dynamo_table_prefix"):
            self.coordinator = StrategyCoordinator(
                table_prefix=config["dynamo_table_prefix"],
                region=config.get("aws_region", "eu-west-1"),
            )
        else:
            self.coordinator = None

    def run(self):
        self.data_adapter.connect()
        self.exec_adapter.connect()

        # Register in DynamoDB
        if self.coordinator:
            self.coordinator.register_strategy(
                strategy_id=self.signal_engine.strategy_id,
                config={"symbol": "EURUSD", "timeframe": "1h"},
            )

        # Recovery...
        if isinstance(self.cache, PersistentCache):
            self.cache.rehydrate()
            self._reconcile_with_broker()

        # Check global locks before starting
        if self.coordinator and self.coordinator.is_trading_frozen():
            print("Trading is globally frozen. Waiting...")
            # Wait or exit depending on policy

        try:
            # Main loop
            while self.data_adapter.has_more_data or not self.bus.is_empty:
                if self.bus.is_empty:
                    bar = self.data_adapter.get_next_bar()
                    if bar:
                        if self.journal:
                            self.journal.record("event.bar", bar)
                        self.bus.publish("event.bar", bar)
                else:
                    self.bus.dispatch_next()
        finally:
            # Deregister on shutdown
            if self.coordinator:
                self.coordinator.deregister_strategy(self.signal_engine.strategy_id)

    def _on_fill(self, fill: FillEvent):
        # ...existing fill handling (update Cache, etc.)...

        # Update exposure in DynamoDB after every fill
        if self.coordinator:
            positions = self.cache.get_positions(
                strategy_id=fill.strategy_id,
                symbol=fill.symbol,
            )
            total_volume = sum(p.volume for p in positions)
            self.coordinator.update_exposure(
                strategy_id=fill.strategy_id,
                symbol=fill.symbol,
                volume=total_volume,
            )
```

### S3 Trade Archiver

```python
class S3TradeArchiver:
    """
    Writes completed trades to S3 as Parquet.
    Partitioned by date and strategy for efficient Athena queries.
    """

    def __init__(self, s3_bucket: str, prefix: str = "trades"):
        self._bucket = s3_bucket
        self._prefix = prefix
        self._buffer = []
        self._flush_interval = 100  # flush every N trades

    def archive(self, fill: FillEvent):
        self._buffer.append({
            "deal": fill.deal.value,
            "symbol": fill.symbol,
            "position_id": fill.position_id,
            "strategy_id": fill.strategy_id,
            "volume": str(fill.volume),
            "price": str(fill.price),
            "signal_type": fill.signal_type.value,
            "commission": str(fill.commission),
            "swap": str(fill.swap),
            "gross_profit": str(fill.gross_profit),
            "timestamp": fill.time_generated.isoformat(),
        })

        if len(self._buffer) >= self._flush_interval:
            self._flush()

    def _flush(self):
        if not self._buffer:
            return
        import polars as pl
        import boto3
        from io import BytesIO

        df = pl.DataFrame(self._buffer)
        buffer = BytesIO()
        df.write_parquet(buffer)
        buffer.seek(0)

        strategy_id = self._buffer[0]["strategy_id"]
        date = datetime.now().strftime("%Y-%m-%d")
        ts = datetime.now().strftime("%H%M%S")
        key = f"{self._prefix}/strategy_id={strategy_id}/date={date}/{ts}.parquet"

        s3 = boto3.client("s3")
        s3.upload_fileobj(buffer, self._bucket, key)
        self._buffer.clear()

    def close(self):
        self._flush()
```

---

## 35.5 Kernel Startup — Container Recovery Flow

```python
class Kernel:
    def __init__(self, config: dict):
        # ...existing init...

        # Use persistent implementations for containerized deployment
        if config.get("redis_url"):
            self.cache = PersistentCache(
                redis_url=config["redis_url"],
                prefix=config.get("redis_prefix", "trading"),
            )
            self.journal = RedisEventJournal(
                redis_url=config["redis_url"],
                stream_key=f"{config.get('redis_prefix', 'trading')}:events",
            )
        else:
            # Fallback: in-memory for backtest
            self.cache = Cache()
            self.journal = None

    def run(self):
        """Main event loop with container recovery."""

        # ── Step 1: Connect adapters ──
        self.data_adapter.connect()
        self.exec_adapter.connect()

        # ── Step 2: Recover state from Redis ──
        if isinstance(self.cache, PersistentCache):
            self.cache.rehydrate()
            print(f"Recovered: {len(self.cache.get_positions())} positions, "
                  f"balance={self.cache.account.balance}")

            # ── Step 3: Reconcile with broker ──
            # After recovery, verify Redis state matches broker state
            self._reconcile_with_broker()

        # ── Step 4: Normal event loop ──
        while self.data_adapter.has_more_data or not self.bus.is_empty:
            if self.bus.is_empty:
                bar = self.data_adapter.get_next_bar()
                if bar:
                    if self.journal:
                        self.journal.record("event.bar", bar)
                    self.bus.publish("event.bar", bar)
            else:
                self.bus.dispatch_next()

    def _reconcile_with_broker(self):
        """
        After recovering state from Redis, compare with broker's actual positions.
        Fix any discrepancies (e.g., a position was closed while container was down).
        """
        broker_positions = self.exec_adapter.get_open_positions()
        cached_positions = {p.ticket: p for p in self.cache.get_positions()}

        broker_tickets = {p["ticket"] for p in broker_positions}
        cached_tickets = set(cached_positions.keys())

        # Positions closed while we were down
        for ticket in cached_tickets - broker_tickets:
            print(f"Reconcile: position {ticket} closed while offline, removing from cache")
            self.cache.remove_position(ticket)

        # Positions opened while we were down (e.g., pending order triggered)
        for pos in broker_positions:
            if pos["ticket"] not in cached_tickets:
                print(f"Reconcile: new position {pos['ticket']} found, adding to cache")
                self.cache.update_position(PositionSnapshot(
                    ticket=pos["ticket"],
                    symbol=pos["symbol"],
                    direction=pos["direction"],
                    volume=Decimal(str(pos["volume"])),
                    price_entry=Decimal(str(pos["price_entry"])),
                    unrealized_pnl=Decimal("0"),
                    strategy_id=pos.get("strategy_id", ""),
                ))

        # Refresh account state from broker
        self.cache.update_account(
            balance=self.exec_adapter.get_balance(),
            equity=self.exec_adapter.get_equity(),
            margin=self.exec_adapter.get_used_margin(),
            margin_free=self.exec_adapter.get_free_margin(),
        )
```

---

## 35.6 ECS Deployment Architecture

### Single-Strategy Setup (ECS Fargate or Lightsail)

```
┌─────────────────────────────────────────────────────────────────┐
│                         AWS VPC                                  │
│                                                                  │
│  ┌──────────────────────┐    ┌────────────────────────────┐     │
│  │ ECS Service:          │    │ ElastiCache (Redis)         │     │
│  │ Strategy Core         │───▶│                             │     │
│  │                       │    │ Hot state (per-strategy):   │     │
│  │ - Kernel              │    │   strategy_a:account        │     │
│  │ - MessageBus          │    │   strategy_a:positions:*    │     │
│  │ - PersistentCache     │    │   strategy_a:pending_orders:*│    │
│  │ - SignalEngine        │    │   strategy_a:events (stream)│     │
│  │ - SizingEngine        │    │   strategy_a:last_sequence  │     │
│  │ - RiskEngine          │    └────────────────────────────┘     │
│  │ - DataAdapter         │                                       │
│  │ - ExecutionAdapter    │    ┌────────────────────────────┐     │
│  │                       │    │ DynamoDB                    │     │
│  │ Env vars:             │───▶│                             │     │
│  │   REDIS_URL           │    │ Shared coordination:        │     │
│  │   DYNAMO_TABLE_PREFIX │    │   strategies (status, config)│    │
│  │   S3_BUCKET           │    │   exposure (per symbol)     │     │
│  │   BROKER_API_KEY      │    │   locks (trading freezes)   │     │
│  └──────────────────────┘    │   alerts (history)          │     │
│                               └────────────────────────────┘     │
│  ┌──────────────────────┐                                       │
│  │ ECS Service:          │    ┌────────────────────────────┐     │
│  │ Monitoring            │    │ S3                          │     │
│  │                       │───▶│                             │     │
│  │ - TradeArchiver       │    │ Cold storage:               │     │
│  │ - Post-trade risk     │    │   trades/ (Parquet)         │     │
│  │ - Alerting            │    │   events/ (journal archive) │     │
│  └──────────────────────┘    │   backtests/ (results)      │     │
│                               │   data/ (historical bars)   │     │
│                               │   snapshots/ (Cache dumps)  │     │
│  ┌──────────────────────┐    └──────────────┬───────────────┘    │
│  │ Secrets Manager       │                   │                    │
│  │                       │           ┌───────▼───────┐           │
│  │ - Broker API keys     │           │ Athena         │           │
│  │ - Redis password      │           │ Cross-strategy │           │
│  └──────────────────────┘           │ reporting SQL  │           │
│                                      └───────────────┘           │
└─────────────────────────────────────────────────────────────────┘
```

### Multi-Strategy Setup

```
┌──────────────────────┐  ┌──────────────────────┐  ┌──────────────────────┐
│ ECS Task: Strategy A  │  │ ECS Task: Strategy B  │  │ ECS Task: Strategy C  │
│ prefix=strategy_a     │  │ prefix=strategy_b     │  │ prefix=strategy_c     │
│ EURUSD MA crossover   │  │ BTCUSDT momentum      │  │ GBPUSD mean reversion │
└──────────┬───────────┘  └──────────┬───────────┘  └──────────┬───────────┘
           │                          │                          │
           └──────────────┬───────────┴──────────────────────────┘
                          │
                 ┌────────▼────────┐
                 │  ElastiCache     │  ← per-strategy isolation by prefix
                 │  (Redis)         │
                 │  strategy_a:*    │
                 │  strategy_b:*    │
                 │  strategy_c:*    │
                 └────────┬────────┘
                          │
                 ┌────────▼────────┐
                 │  DynamoDB        │  ← shared coordination across strategies
                 │                  │
                 │  strategies:     │  strategy_a → running, last_heartbeat=...
                 │                  │  strategy_b → running, last_heartbeat=...
                 │                  │
                 │  exposure:       │  EURUSD → 0.3 lots (a:0.1, c:0.2)
                 │                  │  BTCUSDT → 0.5 lots (b:0.5)
                 │                  │
                 │  locks:          │  freeze_trading → false
                 │                  │  max_portfolio_dd → -5%
                 └────────┬────────┘
                          │
                 ┌────────▼────────┐
                 │  S3 + Athena     │  ← cold storage + cross-strategy reporting
                 │  trades/         │
                 │  events/         │
                 └─────────────────┘
```

Each strategy has isolated hot state in Redis. Shared coordination (which strategies exist, total exposure, global locks) lives in DynamoDB. All historical data and reporting goes to S3/Parquet queryable via Athena.

### Dockerfile

```dockerfile
FROM python:3.12-slim

WORKDIR /app

# Install dependencies
COPY pyproject.toml poetry.lock ./
RUN pip install poetry && poetry install --no-dev --no-interaction

# Copy application code
COPY . .

# Configuration via environment variables
ENV REDIS_URL=redis://localhost:6379
ENV REDIS_PREFIX=trading
ENV STRATEGY_CONFIG=config/strategy.yaml
ENV LOG_LEVEL=INFO

# Health check
HEALTHCHECK --interval=30s --timeout=5s \
    CMD python -c "import redis; r=redis.from_url('${REDIS_URL}'); r.ping()"

ENTRYPOINT ["python", "-m", "trading_engine"]
```

### ECS Task Definition (key parts)

```json
{
    "family": "trading-strategy",
    "networkMode": "awsvpc",
    "requiresCompatibilities": ["FARGATE"],
    "cpu": "512",
    "memory": "1024",
    "containerDefinitions": [
        {
            "name": "strategy-core",
            "image": "your-ecr-repo/trading-engine:latest",
            "essential": true,
            "environment": [
                {"name": "REDIS_PREFIX", "value": "strategy_eurusd_ma"}
            ],
            "secrets": [
                {"name": "REDIS_URL", "valueFrom": "arn:aws:secretsmanager:...redis-url"},
                {"name": "BROKER_API_KEY", "valueFrom": "arn:aws:secretsmanager:...broker-key"}
            ],
            "logConfiguration": {
                "logDriver": "awslogs",
                "options": {
                    "awslogs-group": "/ecs/trading-strategy",
                    "awslogs-region": "eu-west-1",
                    "awslogs-stream-prefix": "strategy"
                }
            },
            "healthCheck": {
                "command": ["CMD-SHELL", "python -c 'import redis; redis.from_url(\"$REDIS_URL\").ping()'"],
                "interval": 30,
                "timeout": 5,
                "retries": 3,
                "startPeriod": 10
            }
        }
    ]
}
```

---

## 35.7 Lightsail Alternative (Simpler, Lower Cost)

For a single strategy, Lightsail is simpler than ECS:

| Component | Lightsail Service | Monthly Cost (approx) |
|---|---|---|
| Strategy container | Lightsail Container Service (micro: 0.25 vCPU, 512MB) | $7/mo |
| Redis | Lightsail Database (Redis, micro) | $15/mo |
| PostgreSQL | Lightsail Database (PostgreSQL, micro) | $15/mo |
| **Total** | | **~$37/mo** |

vs ECS Fargate:

| Component | AWS Service | Monthly Cost (approx) |
|---|---|---|
| Strategy container | Fargate (0.25 vCPU, 512MB, 24/7) | ~$9/mo |
| Redis | ElastiCache (cache.t3.micro) | ~$12/mo |
| PostgreSQL | RDS (db.t3.micro) | ~$15/mo |
| **Total** | | **~$36/mo** |

Cost is similar. Lightsail is simpler to set up. ECS is more flexible for multi-strategy scaling.

---

## 35.8 MT5 Constraint

MetaTrader 5 requires Windows with a GUI. It cannot run inside a Linux container. Options:

| Approach | How it works | Complexity |
|---|---|---|
| **MT5 on Windows EC2** | Run MT5 terminal on a Windows EC2 instance. Your Linux ECS container connects via MT5 Python API over the network. | Medium |
| **MT5 Web API** | If your broker supports MT5 Web API, connect directly from Linux. | Low (if available) |
| **Wine + headless X** | Run MT5 in Wine inside a Linux container with Xvfb virtual display. | High (fragile) |
| **MT5 bridge service** | Dedicated Windows service that wraps MT5 API and exposes a REST/gRPC/ZeroMQ interface. Your Linux containers talk to the bridge. | Medium-High |

The recommended approach for ECS: **MT5 bridge on Windows EC2 + strategy logic in Linux ECS containers**.

```
ECS (Linux, Fargate)              EC2 (Windows)
┌──────────────────┐              ┌──────────────────┐
│ Strategy Core     │──ZeroMQ────▶│ MT5 Bridge        │
│ Kernel            │              │ - MT5 Terminal    │
│ ExecutionAdapter  │◀─ZeroMQ─────│ - REST/ZMQ API    │
│  (ZMQ client)     │              │ - Wraps mt5 pkg   │
└──────────────────┘              └──────────────────┘
```

The `ExecutionAdapter` for this setup would be a ZeroMQ client that sends orders to the MT5 bridge and receives fills back. Same `IExecutionAdapter` interface — the Kernel doesn't know or care that MT5 is on a different machine.

---

## 35.9 Backtest vs Live: Same Code, Different Cache

The beauty of the `ICache` contract is that backtest and live use the same Kernel code:

```python
# Backtest — in-memory, no Redis needed
kernel = Kernel(
    cache=Cache(),                          # plain in-memory
    journal=EventJournal("backtest.jsonl"),  # local file
    data_adapter=CSVDataAdapter(...),
    exec_adapter=SimulatorExecutionAdapter(...),
    ...
)

# Live — Redis-backed, containerized
kernel = Kernel(
    cache=PersistentCache(redis_url=os.environ["REDIS_URL"]),  # write-through
    journal=RedisEventJournal(redis_url=os.environ["REDIS_URL"]),  # Redis Stream
    data_adapter=MT5DataAdapter(...),
    exec_adapter=MT5ExecutionAdapter(...),
    ...
)
```

Same Kernel, same MessageBus, same engines. Only the infrastructure layer changes.

---

## 35.10 Component Map — What Changes for Deployment

| Component | Backtest (local/Airflow) | Live (ECS/Lightsail) |
|---|---|---|
| Cache | `Cache` (in-memory) | `PersistentCache` (in-memory + Redis write-through) |
| Event Journal | `EventJournal` (local JSONL file) | `DurableEventJournal` (Redis Streams + S3 Parquet flush) |
| Coordinator | None (single process) | `StrategyCoordinator` (DynamoDB — strategy registry, exposure, locks) |
| Trade Archiver | `TradeArchiver` (in-memory list) | `S3TradeArchiver` (Parquet to S3, queryable via Athena) |
| Snapshot Manager | `SnapshotManager` (local JSON files) | S3-backed snapshots |
| Data Adapter | `CSVDataAdapter` (local or S3) | `MT5DataAdapter` / `BinanceDataAdapter` / etc. |
| Execution Adapter | `SimulatorExecutionAdapter` | `MT5ExecutionAdapter` / `IBKRAdapter` / etc. |
| Kernel | Same | Same (+ `rehydrate()` + `reconcile()` + `register/deregister` on startup/shutdown) |
| MessageBus | Same | Same |
| SignalEngine | Same | Same |
| SizingEngine | Same | Same |
| RiskEngine | Same | Same |

### Storage mapping

| Store | Local/Dev | Prod (AWS) | Purpose |
|---|---|---|---|
| **Hot state** | Redis (Docker) | ElastiCache Redis | Account, positions, orders, event stream |
| **Shared coordination** | (not needed, single process) | DynamoDB | Strategy registry, cross-strategy exposure, global locks |
| **Cold storage** | Local files | S3 + Athena | Trade history, event archive, backtest results, reporting |
| **Airflow metadata** | PostgreSQL (Docker) | (not used by trading engine) | Airflow internal only |

**The trading engine never touches PostgreSQL.** PostgreSQL only exists locally because Airflow needs it.

---
