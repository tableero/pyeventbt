# Package: `pyeventbt.events`

## Purpose

Defines the core event types and data structures that flow through the PyEventBT event-driven architecture. Every component in the system communicates by placing typed event objects onto a shared `queue.Queue`; this package provides those event definitions along with supporting enumerations and the `Bar` data payload.

## Tags

`events`, `core`, `data-model`, `pydantic`, `dataclass`, `enum`

## Modules

| Module | File | Description |
|---|---|---|
| `events` | `events.py` | All event types (`BarEvent`, `SignalEvent`, `OrderEvent`, `FillEvent`, `ScheduledEvent`), supporting enums (`EventType`, `SignalType`, `OrderType`, `DealType`), and the `Bar` dataclass payload |
| `__init__` | `__init__.py` | Re-exports all symbols from `events.py` via wildcard import; also re-exports `SuggestedOrder` from `pyeventbt.portfolio_handler.core.entities.suggested_order` |

## Internal Architecture

The package is flat -- a single module (`events.py`) contains all definitions. The `__init__.py` performs a star-import so consumers can write `from pyeventbt.events import BarEvent` directly.

### Design Decisions

- **`Bar` as a slotted dataclass**: Chosen for memory compactness (~56 bytes). Prices are stored as integers with a `digits` field that records how many decimal places to divide by to reconstruct floats. This avoids floating-point overhead during bulk data storage.
- **`EventBase` as a Pydantic `BaseModel`**: Provides validation and serialization. The `Config.arbitrary_types_allowed = True` setting is needed because some events carry `pd.Timestamp` fields.
- **Enums inherit from `(str, Enum)`**: This makes them JSON-serializable and usable as dictionary keys without extra conversion.

### Event Lifecycle

```
DataProvider --> BarEvent --> TradingDirector
                                |
                    SignalEngineService --> SignalEvent --> TradingDirector
                                                             |
                                              PortfolioHandler --> OrderEvent --> TradingDirector
                                                                                    |
                                                                     ExecutionEngine --> FillEvent --> TradingDirector
                                                                                                         |
                                                                                          PortfolioHandler (updates state)
```

## Cross-Package Dependencies

| Dependency | Usage |
|---|---|
| `pyeventbt.strategy.core.strategy_timeframes.StrategyTimeframes` | Used by `ScheduledEvent.schedule_timeframe` |
| `pyeventbt.portfolio_handler.core.entities.suggested_order.SuggestedOrder` | Re-exported from `__init__.py` |
| `pydantic` | `BaseModel` for all event classes except `Bar` |
| `pandas` | `pd.Timestamp` used in `ScheduledEvent` |

## Gaps & Issues

1. **`SuggestedOrder` re-export is a coupling leak**: The `events/__init__.py` re-exports `SuggestedOrder` from the `portfolio_handler` package. This entity is not an event and its presence here is likely for convenience but creates an unexpected cross-package dependency.
2. **`Bar.__price_factor` typing**: The private field defaults to `None` but is typed as `float` in the `field()` declaration. The actual runtime type is `Optional[float]` until first access.
3. **Typo in comment**: Line 51 of `events.py` reads "comppact" and "arount" (should be "compact" and "around").
4. **`rollover` field uses a plain tuple**: The `(False, "", "")` structure would benefit from a named type or dataclass for clarity. The semantics (flag, original_contract, new_contract) are only documented in a comment.
5. **No `__all__` defined in `events.py`**: The wildcard import in `__init__.py` will export everything including imports like `BaseModel`, `Decimal`, `pd`, etc.
