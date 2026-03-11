# pyeventbt.indicators.core.interfaces.indicator_interface

- **File**: `pyeventbt/indicators/core/interfaces/indicator_interface.py`
- **Module**: `pyeventbt.indicators.core.interfaces.indicator_interface`
- **Purpose**: Defines the abstract interface (`IIndicator`) that all technical indicator classes must implement.
- **Tags**: `interface`, `abc`, `indicators`, `contract`

## Dependencies

| Dependency | Usage |
|---|---|
| `abc.ABC` | Base class for abstract interface |
| `abc.abstractmethod` | Imported but not applied to `compute()` (see Gaps) |
| `pandas` (`pd`) | Used in the interface signature for `pd.DataFrame` and `pd.Series` types |

Note: `abc` and `pandas` are each imported twice (duplicate import statements at lines 11-12 and 14-15).

## Classes/Functions

### IIndicator

- **Signature**: `class IIndicator(ABC)`
- **Description**: Abstract base class defining the contract for indicator implementations. All concrete indicators (KAMA, SMA, EMA, ATR, RSI, etc.) inherit from this class.

#### Methods

| Method | Signature | Description | Returns |
|---|---|---|---|
| `compute` | `compute(self, data: pd.DataFrame) -> pd.Series` | Declared as `@staticmethod` but includes `self` parameter. Intended to define the contract for calculating indicator values from input data. Body is `pass` (no `raise NotImplementedError`). | `pd.Series` (per signature) |

## Data Flow

1. `IIndicator` is imported by `indicators.py`.
2. Each concrete indicator class (`KAMA`, `SMA`, etc.) inherits from `IIndicator`.
3. Concrete classes override `compute()` with their own static method signatures (which differ from the interface signature).

## Gaps & Issues

1. **`@abstractmethod` imported but not used**: The `abstractmethod` decorator is imported but not applied to `compute()`. This means subclasses are not required by Python to implement `compute()`, defeating the purpose of the ABC.
2. **Signature mismatch with implementations**: The interface declares `compute(self, data: pd.DataFrame) -> pd.Series`, but all concrete implementations use `@staticmethod` with `np.ndarray` parameters and return `np.ndarray` or `tuple`. The interface does not reflect the actual contract.
3. **`@staticmethod` with `self` parameter**: `compute` is decorated with `@staticmethod` but declares a `self` parameter, which is contradictory. Python will not pass `self` to a static method.
4. **Duplicate imports**: Both `abc` (`ABC`, `abstractmethod`) and `pandas` are imported twice on consecutive lines.
5. **Body is `pass` not `raise NotImplementedError`**: The method body does nothing, providing no runtime guard against unimplemented subclasses.

## Requirements Derived

- R-IFACE-01: All indicator classes must inherit from `IIndicator`.
- R-IFACE-02: The `compute()` method should be the single public entry point for indicator calculation.
- R-IFACE-03: The interface signature should be updated to match the actual usage pattern (`@staticmethod` with `np.ndarray` parameters).
