# pyeventbt.indicators.indicators

- **File**: `pyeventbt/indicators/indicators.py`
- **Module**: `pyeventbt.indicators.indicators`
- **Purpose**: Implements all Numba-accelerated technical indicators. Each indicator is a class with a private `@njit` computation kernel and a public static `compute()` method.
- **Tags**: `indicators`, `numba`, `njit`, `technical-analysis`, `performance`

## Dependencies

| Dependency | Usage |
|---|---|
| `numpy` | Array construction, NaN initialization, math operations |
| `numba.njit` | JIT-compiles private computation methods for performance |
| `pandas.DataFrame`, `pandas.Series` | Imported but unused |
| `.core.interfaces.indicator_interface.IIndicator` | Base class for all indicators |

## Classes/Functions

### KAMA

- **Signature**: `KAMA.compute(close: np.ndarray, n_period: int = 10, period_fast: int = 2, period_slow: int = 30) -> np.ndarray`
- **Description**: Kaufman Adaptive Moving Average. Adjusts smoothing based on market efficiency ratio. Raises `ValueError` if `len(close) < n_period`.
- **Attributes**: None (stateless static class).
- **Returns**: `np.ndarray` of KAMA values; indices before `n_period - 1` are `NaN`.
- **Private kernel**: `__compute_kama(close, n_period, sc_fastest, sc_slowest)` -- `@njit`.

### SMA

- **Signature**: `SMA.compute(close: np.ndarray, period: int) -> np.ndarray`
- **Description**: Simple Moving Average using a rolling sum approach. Raises `ValueError` if `len(close) < period`.
- **Returns**: `np.ndarray` of SMA values; indices before `period - 1` are `NaN`.
- **Private kernel**: `__compute_sma(close, period)` -- `@njit`.

### EMA

- **Signature**: `EMA.compute(close: np.ndarray, period: int) -> np.ndarray`
- **Description**: Exponential Moving Average. Initializes with SMA of first `period` values, then applies EMA multiplier `2 / (period + 1)`. Raises `ValueError` if `len(close) < period`.
- **Returns**: `np.ndarray` of EMA values; indices before `period - 1` are `NaN`.
- **Private kernel**: `__compute_ema(close, period)` -- `@njit`.

### ATR

- **Signature**: `ATR.compute(high: np.ndarray, low: np.ndarray, close: np.ndarray, period: int, method: str = 'sma') -> np.ndarray`
- **Description**: Average True Range. Computes True Range first, then smooths via SMA or EMA. Raises `ValueError` if arrays differ in length or `method` is invalid.
- **Returns**: `np.ndarray` of ATR values; indices before `period - 1` are `NaN`.
- **Private kernels**: `__compute_tr(high, low, close)`, `__compute_atr_sma(tr, period)`, `__compute_atr_ema(tr, period)` -- all `@njit`.

### RSI

- **Signature**: `RSI.compute(close: np.ndarray, period: int = 14) -> np.ndarray`
- **Description**: Relative Strength Index using Wilder's smoothed average method. Raises `ValueError` if `len(close) < period + 1`.
- **Returns**: `np.ndarray` of RSI values (0-100 range); indices before `period` are `NaN`.
- **Private kernel**: `__compute_rsi(close, period)` -- `@njit`.

### ADX

- **Signature**: `ADX.compute(high: np.ndarray, low: np.ndarray, close: np.ndarray, period: int = 14) -> tuple`
- **Description**: Average Directional Index with +DI and -DI. Raises `ValueError` if arrays differ in length or `len < period * 2`.
- **Returns**: `tuple[np.ndarray, np.ndarray, np.ndarray]` -- `(adx, plus_di, minus_di)`.
- **Private kernel**: `__compute_adx(high, low, close, period)` -- `@njit`.

### Momentum

- **Signature**: `Momentum.compute(close: np.ndarray, period: int = 10) -> np.ndarray`
- **Description**: Simple price momentum (difference between current close and close `period` bars ago). Raises `ValueError` if `len(close) < period + 1`.
- **Returns**: `np.ndarray` of momentum values; indices before `period` are `NaN`.
- **Private kernel**: `__compute_momentum(close, period)` -- `@njit`.

### BollingerBands

- **Signature**: `BollingerBands.compute(close: np.ndarray, period: int = 20, std_dev: float = 2.0) -> tuple`
- **Description**: Bollinger Bands with SMA middle line and standard deviation bands. Uses population standard deviation (`variance / period`). Raises `ValueError` if `len(close) < period`.
- **Returns**: `tuple[np.ndarray, np.ndarray, np.ndarray]` -- `(upper_band, middle_band, lower_band)`.
- **Private kernel**: `__compute_bollinger(close, period, std_dev)` -- `@njit`.

### DonchianChannels

- **Signature**: `DonchianChannels.compute(high: np.ndarray, low: np.ndarray, period: int = 20) -> tuple`
- **Description**: Donchian Channels (highest high, lowest low, midpoint over period). Raises `ValueError` if arrays differ in length or `len < period`.
- **Returns**: `tuple[np.ndarray, np.ndarray, np.ndarray]` -- `(upper, middle, lower)`.
- **Private kernel**: `__compute_donchian(high, low, period)` -- `@njit`.

### MACD

- **Signature**: `MACD.compute(close: np.ndarray, fast_period: int = 12, slow_period: int = 26, signal_period: int = 9) -> tuple`
- **Description**: Moving Average Convergence Divergence. Computes fast/slow EMAs, MACD line, signal line (EMA of MACD), and histogram. Raises `ValueError` if `len(close) < slow_period + signal_period`.
- **Returns**: `tuple[np.ndarray, np.ndarray, np.ndarray]` -- `(macd_line, signal_line, histogram)`.
- **Private kernels**: `__compute_ema_for_macd(close, period)`, `__compute_macd(close, fast_period, slow_period, signal_period)` -- `@njit`.

### KeltnerChannel

- **Signature**: `KeltnerChannel.compute(high: np.ndarray, low: np.ndarray, close: np.ndarray, period: int = 20, atr_period: int = 10, multiplier: float = 2.0) -> tuple`
- **Description**: Keltner Channel using EMA middle line and ATR-based bands. Raises `ValueError` if arrays differ in length or `len < max(period, atr_period)`.
- **Returns**: `tuple[np.ndarray, np.ndarray, np.ndarray]` -- `(upper, middle, lower)`.
- **Private kernel**: `__compute_keltner(high, low, close, period, atr_period, multiplier)` -- `@njit`.

## Data Flow

1. User calls `IndicatorClass.compute(...)` with numpy arrays (typically extracted from polars DataFrames via `.to_numpy().flatten()`).
2. `compute()` validates input lengths and parameters.
3. `compute()` delegates to one or more private `@njit` methods via Python name mangling.
4. The `@njit` method performs the numerical computation in compiled machine code.
5. Result (`np.ndarray` or `tuple` of arrays) is returned to the caller.

## Gaps & Issues

1. **Unused pandas imports**: `DataFrame` and `Series` are imported at the top of the module but never used.
2. **Interface deviation**: All classes inherit `IIndicator` but none match its declared `compute(self, data: pd.DataFrame) -> pd.Series` signature. The actual signatures use `@staticmethod` with `np.ndarray` I/O.
3. **BollingerBands uses population std dev**: The variance is divided by `period` (not `period - 1`), which gives population standard deviation rather than sample standard deviation. This may or may not be intentional.
4. **Inconsistent export**: Only `KAMA`, `SMA`, `EMA`, `ATR` are exported from `__init__.py`; `RSI`, `ADX`, `Momentum`, `BollingerBands`, `DonchianChannels`, `MACD`, `KeltnerChannel` are not.

## Requirements Derived

- R-IND-01: All indicators must accept numpy arrays and return numpy arrays or tuples of numpy arrays.
- R-IND-02: Indicators must raise `ValueError` for insufficient input data length.
- R-IND-03: NaN values must fill positions where the indicator cannot yet be computed (warm-up period).
- R-IND-04: Numba `@njit` must be applied to inner computation loops for performance.
- R-IND-05: Each indicator must implement `IIndicator` (via static `compute()` method).
