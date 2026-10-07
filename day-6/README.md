# Day 6 — Passing `ruff` and `black` Code-Quality Checks

## Overview

Part of the *100 Days of MLOps* challenge. This exercise reproduces a failing code-quality
gate, explains what `ruff` and `black` actually are, and walks through the configuration
and source changes needed to make both tools pass.

The scenario: the xFusionCorp Industries ML team enforces code-quality standards using
`ruff` and `black` on every pull request. The project at `/root/code/fraud-detection/`
is failing both tools and must be brought back to a passing state.

> **Note** — The `pyproject.toml` shown in
> [Initial state](#initial-state-the-failing-project) is the original, **non-compliant**
> version, kept on purpose to document the failing starting point. The corrected version
> is shown in the [Solution](#solution) section.

## Task

The end state must satisfy the following:

- `ruff` and `black` are both configured with a line length of **120**.
- The `ruff` lint rule selection includes **`E`, `F`, `W`, and `I`**.
- Running `ruff check src/` from the project directory exits with status **0**.
- Running `black --check src/` from the project directory exits with status **0**.

## The tools: what `ruff` and `black` are

They are two complementary Python code-quality tools, commonly wired into CI/CD (GitHub
Actions, pre-commit hooks, etc.). They are *not* alternatives to each other.

### `black` — the formatter

- An **automatic code formatter**: it does not report problems, it **rewrites the files**
  into a single, consistent style.
- Famous tagline: *"The uncompromising code formatter"* — very few options, no style
  debates between teammates.
- Configured almost exclusively through `line-length` (default `88`).
- `black --check src/` **changes nothing**: it only verifies whether files *would* be
  reformatted and returns a non-zero exit code if so (ideal for CI). To actually apply the
  changes, run `black src/` without `--check`.

### `ruff` — the linter (and also a formatter)

- A **linter** written in **Rust**, extremely fast. It was born as an all-in-one
  replacement for historical tools such as `flake8`, `isort`, `pyupgrade` and parts of
  `pylint`.
- Performs **static analysis**: it finds unused imports, unused variables, style/syntax
  issues, unsorted imports, and more. Many problems are auto-fixable with
  `ruff check --fix`.
- Configured in `pyproject.toml` via rule codes. The ones used here:
  - **`E`** — pycodestyle *errors* (spacing, line length, indentation, …)
  - **`F`** — pyflakes (e.g. `F401 import ... imported but unused`)
  - **`W`** — pycodestyle *warnings* (e.g. trailing whitespace)
  - **`I`** — isort (import ordering)
- `ruff` also ships a formatter (`ruff format`), but this exercise uses it only as a linter.

### Key difference

| | `black` | `ruff` |
| --- | --- | --- |
| What it does | Reformats the code | Reports problems (lint) |
| Modifies files | Yes (without `--check`) | Only with `--fix` |
| Configuration | `line-length` | `line-length` + rule selection |

## Initial state: the failing project

Running the tools as committed:

```console
$ ruff check src/
warning: The top-level linter settings are deprecated in favour of their counterparts in the `lint` section. Please update the following options in `pyproject.toml`:
  - 'select' -> 'lint.select'
F401 [*] `os` imported but unused
 --> src/data/process_data.py:1:8
  |
1 | import os
  |        ^^
2 | import pandas as pd
  |
help: Remove unused import: `os`
  |
  - import os
1 | import pandas as pd
  |

Found 1 error.
[*] 1 fixable with the `--fix` option.

$ black --check src/
would reformat /root/code/fraud-detection/src/data/process_data.py

Oh no! 💥 💔 💥
1 file would be reformatted, 4 files would be left unchanged.
```

The offending configuration in `pyproject.toml`:

```toml
[project]
name = "fraud-detection"
version = "0.1.0"

[tool.ruff]
line-length = 88
select = ["E", "F"]

[tool.black]
line-length = 100
```

Two kinds of problems:

| Requirement | Initial state |
| --- | --- |
| `ruff` line length = 120 | `line-length = 88` |
| `black` line length = 120 | `line-length = 100` (also inconsistent with ruff) |
| `ruff` rules include `E, F, W, I` | `select = ["E", "F"]` — `W` and `I` missing |
| No deprecation warning | `select` sits at the top level of `[tool.ruff]` |
| `ruff check src/` exits 0 | `F401` — unused `import os` in `src/data/process_data.py` |
| `black --check src/` exits 0 | one file would be reformatted |

## Solution

### 1. `pyproject.toml`

Align both tools on `line-length = 120`, add the missing rules, and move `select` into the
`[tool.ruff.lint]` sub-section to silence the deprecation warning:

```toml
[project]
name = "fraud-detection"
version = "0.1.0"

[tool.ruff]
line-length = 120

[tool.ruff.lint]
select = ["E", "F", "W", "I"]

[tool.black]
line-length = 120
```

Two details matter here:

- `line-length` stays under `[tool.ruff]` because it is a **general** option, not a lint
  one. Putting it under `[tool.ruff.lint]` makes `ruff` fail with
  `unknown field 'line-length'`.
- `select` moves under `[tool.ruff.lint]` because it *is* a lint option. The old top-level
  location still works for backwards compatibility, but emits the deprecation warning.

### 2. Source code

Fix the `F401` error and let `black` reformat the file:

```bash
ruff check --fix src/   # removes the unused `import os` (and sorts imports)
black src/              # reformats the files in place
```

## Verification

Run the following from the project directory to confirm the end state.

**1. `ruff` passes with no warning**

```bash
ruff check src/         # expected: "All checks passed!" and exit code 0
```

**2. `black` reports nothing to change**

```bash
black --check src/      # expected: "5 files would be left unchanged." and exit code 0
```

**3. Confirm the configuration**

```bash
grep -A2 '\[tool.ruff\]'      pyproject.toml   # line-length = 120
grep -A2 '\[tool.ruff.lint\]' pyproject.toml   # select = ["E", "F", "W", "I"]
grep -A2 '\[tool.black\]'     pyproject.toml   # line-length = 120
```

Expected final output:

```console
$ ruff check src/
All checks passed!

$ black --check src/
All done! ✨ 🍰 ✨
5 files would be left unchanged.
```

## Repository notes

- This directory documents the exercise; the project itself lives at
  `/root/code/fraud-detection/` and is not committed here.
- `task.txt` is the exercise brief and is kept local only (see `.gitignore`).
