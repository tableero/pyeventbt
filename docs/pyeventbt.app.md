# Module: pyeventbt.app

**File**: `pyeventbt/app.py`
**Module**: `pyeventbt.app`
**Purpose**: CLI entry point for the `pyeventbt` command. Provides version display and basic framework information.
**Tags**: `#cli` `#entry-point` `#argparse`

---

## Dependencies

| Import | Source |
|---|---|
| `argparse` | Standard library |
| `sys` | Standard library |
| `__version__` | `pyeventbt` (relative import) |

---

## Functions

### `main()`

```python
def main() -> None
```

**Description**: Main entry point for the PyEventBT CLI, registered as `pyeventbt` console script in `pyproject.toml`. Parses command-line arguments using `argparse`.

**Behavior**:
- `--version` flag: prints `PyEventBT {version}` and exits
- `info` command (or no command): calls `print_info()`

**Arguments parsed**:
| Argument | Type | Description |
|---|---|---|
| `--version` | flag | Prints version and exits |
| `command` | positional, optional | Choices: `["info"]`. Defaults to `None` (same as `info`) |

**Returns**: `None`

---

### `print_info()`

```python
def print_info() -> None
```

**Description**: Prints a formatted information banner including version, documentation URL, GitHub URL, and a usage hint showing the basic import statement.

**Output format**:
```
PyEventBT v{version}
========================================
Documentation: https://pyeventbt.com
GitHub:        https://github.com/marticastany/pyeventbt
========================================

PyEventBT is a framework/library.
To use it in your project, import the components:

    from pyeventbt import Strategy, BarEvent, SignalEvent

For examples and tutorials, visit the documentation.
```

**Returns**: `None`

---

## Data Flow

- **Inbound**: Invoked by the Poetry-generated console script `pyeventbt` which calls `main()`.
- **Outbound**: Writes to stdout only. No side effects, no file I/O, no network calls.

---

## Gaps & Issues

1. **`sys` imported but unused**: The `sys` module is imported but never referenced in the code.
2. **Limited CLI functionality**: The CLI only supports `--version` and `info`. There is no command to run a strategy, validate configuration, or perform other framework operations from the command line.
3. **No subcommand extensibility**: The argparse setup uses a simple positional argument rather than subparsers, making it harder to extend with additional commands in the future.
4. **Documentation URL may be inactive**: References `https://pyeventbt.com` which may not resolve.

---

## Requirements Derived

- R-APP-01: The `pyeventbt` CLI command must display the current package version when invoked with `--version`.
- R-APP-02: The default CLI behavior (no arguments) must display framework information and usage guidance.
