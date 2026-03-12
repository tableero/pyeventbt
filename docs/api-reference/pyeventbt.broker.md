# Package: `pyeventbt.broker`

## Purpose
Top-level broker abstraction package. Serves as the namespace root for all broker integrations within PyEventBT. Currently contains a single sub-package (`mt5_broker`) implementing the MetaTrader 5 broker interface for both simulated backtesting and live trading.

## Tags
`broker`, `trading`, `mt5`, `namespace`, `top-level`

## Modules

| Module / Sub-package | Description |
|---|---|
| `__init__.py` | Empty init with license header. No public exports. |
| `mt5_broker/` | Full MetaTrader 5 broker implementation (simulator wrapper, live connector, entities, interfaces, shared state). |

## Internal Architecture

```
broker/
  __init__.py                  # Namespace package (no exports)
  mt5_broker/                  # MetaTrader 5 broker implementation
    __init__.py                # Re-exports Mt5SimulatorWrapper facade
    mt5_simulator_wrapper.py   # Drop-in replacement for `import MetaTrader5 as mt5`
    connectors/                # Concrete connector implementations
    core/                      # Interfaces and entity models
    shared/                    # Global mutable state + YAML defaults
```

The `broker` package follows a pluggable architecture pattern. Each broker integration lives in its own sub-package. The `mt5_broker` sub-package is the only implementation at present.

## Cross-Package Dependencies

- **Upstream**: `pyeventbt.utils.utils` (platform compatibility check used by `live_mt5_broker`)
- **Upstream**: `pyeventbt.config` (`Mt5PlatformConfig` used by `LiveMT5Broker`)
- **Downstream consumers**: `pyeventbt.execution_engine` (imports broker connectors to place orders), `pyeventbt.data_provider` (uses broker for live data feeds)

## Gaps & Issues

1. **Single broker implementation** -- Only MetaTrader 5 is supported. No abstract broker interface exists at the `broker/` level to facilitate adding new broker integrations.
2. **Empty `__init__.py`** -- The package init exports nothing, so consumers must import from deeply nested paths or rely on `mt5_broker.__init__` re-exports.
3. **No broker factory** -- There is no registry or factory pattern for selecting a broker at runtime; wiring is done directly in the `Strategy` class.
