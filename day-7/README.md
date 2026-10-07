# Day 7 — Testing and Packaging the Fraud-Detection Module

## Overview

Part of the *100 Days of MLOps* challenge. This exercise adds unit tests to a machine
learning module and turns it into an installable Python distribution, documenting the
packaging configuration, the `src/` layout, and the commands used to verify the end state.

The scenario: the xFusionCorp Industries deployment team requires the fraud-detection
module to be validated through unit tests and to be packaged as an installable Python
distribution. The module source and a draft `pyproject.toml` are found at
`/root/code/fraud-detection/`; the packaging configuration is incomplete and cannot
produce a valid wheel.

> **Note** — The `pyproject.toml` shown in
> [Initial state](#initial-state-the-non-compliant-pyprojecttoml) is the original,
> **non-compliant** version, kept on purpose to document the failing starting point. The
> corrected version is shown in the [Solution](#solution) section.

## Task

The end state must satisfy the following:

- `tests/test_predict.py` contains at least two tests that import `predict` from
  `fraud_detection` and assert on its output — one fraudulent row (amount `> 100`,
  expect `1`) and one legitimate row (amount `<= 100`, expect `0`); `pytest` run from the
  project directory passes.
- The corrected `pyproject.toml` satisfies every one of the following:
  - a `[build-system]` section with `requires = ["setuptools>=61.0", "wheel"]` and
    `build-backend = "setuptools.build_meta"`;
  - `name` is `fraud_detection`;
  - `version` is `0.1.0`;
  - `requires-python` is `>=3.10`;
  - `dependencies` is `["scikit-learn", "pandas", "numpy"]`;
  - `[tool.pytest.ini_options]` declares `pythonpath = ["src"]` so pytest can import the
    package from `src/`.
- Building the package produces a wheel named `fraud_detection-0.1.0-*.whl` under `dist/`.

## Background: `pytest` and Python packaging

### `pytest`

`pytest` discovers files named `test_*.py`, runs every function named `test_*`, and fails
any `assert` that does not hold. Testing a module before shipping it is how you prove it
behaves as intended. The tests import the module from `src/`; the `pythonpath = ["src"]`
setting under `[tool.pytest.ini_options]` tells pytest where to find it, so the package
does not need to be installed first.

### Python packaging

An installable Python distribution is described by `pyproject.toml`:

- `[build-system]` names the build backend (`setuptools` here) and the packages the
  isolated build environment must install.
- `[project]` holds the metadata — `name`, `version`, `requires-python`, and runtime
  `dependencies`.

`python3 -m build` reads this file and writes a *wheel* (`.whl`, the installable binary
format) plus a source distribution (`.tar.gz`) to `dist/`.

### The `src/` layout

The package lives at `src/fraud_detection/`, not at the project root. This is the
*`src` layout*, which prevents accidental imports of the uninstalled package during tests.
`[tool.setuptools.packages.find] where = ["src"]` tells setuptools to look for packages
inside `src/`.

## Initial state: the non-compliant `pyproject.toml`

```toml
[project]
name = "fraud-detection"
version = "0.0.1"
description = "Fraud detection model for xFusionCorp Industries"
requires-python = ">=3.8"
dependencies = []

[tool.setuptools.packages.find]
where = ["src"]
```

| Requirement | Initial state |
| --- | --- |
| `[build-system]` with `setuptools>=61.0`, `wheel`, `setuptools.build_meta` | section missing entirely |
| `name` is `fraud_detection` | `name = "fraud-detection"` (hyphen) |
| `version` is `0.1.0` | `version = "0.0.1"` |
| `requires-python` is `>=3.10` | `requires-python = ">=3.8"` |
| `dependencies` is `["scikit-learn", "pandas", "numpy"]` | `dependencies = []` |
| `pythonpath = ["src"]` for pytest | `[tool.pytest.ini_options]` missing |
| `tests/test_predict.py` exists and passes | `tests/` is empty |

Without a `[build-system]` section, `python3 -m build` does not know which backend to use;
with an empty `tests/` directory, `python3 -m pytest -q` reports `no tests ran`.

## Solution

### 1. Correct `pyproject.toml`

```toml
[build-system]
requires = ["setuptools>=61.0", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "fraud_detection"
version = "0.1.0"
description = "Fraud detection model for xFusionCorp Industries"
requires-python = ">=3.10"
dependencies = [
    "scikit-learn",
    "pandas",
    "numpy",
]

[tool.setuptools.packages.find]
where = ["src"]

[tool.pytest.ini_options]
pythonpath = ["src"]
```

### 2. Write the unit tests

The module under test is a placeholder: `predict()` returns `1` for any row whose first
value (the amount) exceeds `100`, and `0` otherwise.

```python
# src/fraud_detection/predict.py
def predict(data):
    return [1 if row[0] > 100 else 0 for row in data]
```

Because `predict()` accepts an iterable of rows and returns a **list** of labels, the
assertions compare against a list:

```python
# tests/test_predict.py
from fraud_detection import predict


def test_flags_high_amount():
    assert predict([[150.0]]) == [1]


def test_passes_low_amount():
    assert predict([[50.0]]) == [0]
```

`from fraud_detection import predict` works thanks to the re-export in
`src/fraud_detection/__init__.py` (`from .predict import predict`) and the
`pythonpath = ["src"]` setting.

### 3. Build the distribution

```bash
python3 -m build
```

## Verification

Run the following from the project directory (`/root/code/fraud-detection`).

**1. Tests pass**

```bash
python3 -m pytest tests/ -q     # expected: "2 passed"
```

**2. The wheel is present and correctly named**

```bash
ls dist/                        # expected: fraud_detection-0.1.0-*.whl (+ .tar.gz)
```

Expected final output:

```console
$ python3 -m pytest tests/ -q
2 passed in 0.06s

$ ls dist/
fraud_detection-0.1.0-py3-none-any.whl  fraud_detection-0.1.0.tar.gz
```

## Gotchas

### The hyphen/underscore trap

The distribution name must be `fraud_detection` with an **underscore**. Writing
`name = "fraud-detection"` (hyphen) still produces a wheel called
`fraud_detection-0.1.0-py3-none-any.whl`, because the wheel filename normalises hyphens to
underscores (PEP 427 / PEP 503). The built artefact therefore *looks* correct while the
`project.name` field is still wrong — and a validator that reads the raw value will reject
it. Always check the source field, not just the output filename:

```bash
grep -n 'name' pyproject.toml   # must print: name = "fraud_detection"
```

### Other points

- Use `python3`, not `python`, as the task specifies.
- Run `pytest` from the project directory, not from `tests/`, so the `pythonpath` setting
  is picked up.
- `warning: sdist: standard file not found: should have one of README, README.rst,
  README.txt, README.md` is harmless: the wheel is still built. Adding a `README.md` to
  the project silences it but is not required.
- `pythonpath` under `[tool.pytest.ini_options]` requires `pytest >= 7.0`.

## Repository notes

- This directory documents the exercise; the project itself lives at
  `/root/code/fraud-detection/` and is not committed here.
- `task.txt` is the exercise brief and is kept local only (see `.gitignore`).
- Generated build artefacts (`build/`, `dist/`, `src/*.egg-info/`) are not committed.
