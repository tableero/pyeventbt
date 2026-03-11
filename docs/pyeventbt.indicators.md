# pyeventbt.indicators

- **Package**: `pyeventbt.indicators`
- **Purpose**: Provides Numba-accelerated technical indicators for use in trading strategies. Acts as the public entry point for all indicator classes, re-exporting them from the internal `indicators.indicators` module.
- **Tags**: `indicators`, `technical-analysis`, `numba`, `package-init`

## Modules

| Module | Description |
|---|---|
| `indicators.indicators` | Core implementation of all indicator classes (KAMA, SMA, EMA, ATR, RSI, ADX, Momentum, BollingerBands, DonchianChannels, MACD, KeltnerChannel) |
| `indicators.core.interfaces.indicator_interface` | Abstract base interface (`IIndicator`) that all indicator classes implement |

## Internal Architecture

The package follows a layered structure:

1. **Interface layer** (`core/interfaces/indicator_interface.py`): Defines `IIndicator` ABC with a static `compute()` method contract.
2. **Implementation layer** (`indicators.py`): Each indicator is a class implementing `IIndicator`. Every class contains a private `@njit`-decorated computation function and a public static `compute()` method that validates inputs and delegates to the Numba-compiled function.
3. **Package init** (`__init__.py`): Re-exports `KAMA`, `SMA`, `EMA`, and `ATR` via `__all__`. Note that additional indicators (RSI, ADX, Momentum, BollingerBands, DonchianChannels, MACD, KeltnerChannel) exist in `indicators.py` but are **not** re-exported from the package init.

All Numba-compiled private methods use Python name mangling (e.g., `_KAMA__compute_kama`) and are invoked via that mangled name in the public `compute()` methods.

## Cross-Package Dependencies

| Dependency | Usage |
|---|---|
| `numpy` | Array types for all input/output data |
| `numba` (`njit`) | JIT compilation of inner computation loops |
| `pandas` | Imported in `indicators.py` (`DataFrame`, `Series`) but not actively used by any indicator; imported in `indicator_interface.py` for the interface signature |

## Gaps & Issues

1. **Incomplete `__all__` exports**: `__init__.py` only exports `KAMA`, `SMA`, `EMA`, `ATR`. The following indicators are defined in `indicators.py` but not re-exported: `RSI`, `ADX`, `Momentum`, `BollingerBands`, `DonchianChannels`, `MACD`, `KeltnerChannel`.
2. **Interface mismatch**: `IIndicator.compute()` is declared with `(self, data: pd.DataFrame) -> pd.Series` but all concrete implementations use `@staticmethod` with `np.ndarray` parameters and return `np.ndarray` or `tuple`. The interface does not enforce the actual contract.
3. **Unused pandas import**: `pandas.DataFrame` and `pandas.Series` are imported in `indicators.py` but never used by any indicator class.
4. **No BollingerBands dict return**: The CLAUDE.md states `BollingerBands.calculate()` returns a dict with `upper/middle/lower/bandwidth/pct_b`; the actual implementation returns a `tuple` of `(upper, middle, lower)` with no `bandwidth` or `pct_b` arrays.
5. **Method naming inconsistency**: The CLAUDE.md references `calculate()` as the public method, but the actual public method is named `compute()`.
