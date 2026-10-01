# Day 5 — Fixing a Makefile Task Runner (`missing separator`)

## Overview

Part of the *100 Days of MLOps* challenge. This exercise reproduces a classic GNU Make
failure, documents the root cause, and walks through the fix, the rationale behind
`.PHONY`, and the commands used to verify the end state.

The scenario: the xFusionCorp Industries Machine Learning team uses a `Makefile` to
streamline data processing, training, testing and cleanup. A preliminary `Makefile`
found in `/root/code/fraud-detection/` does not complete when running `make all`.

> **Note** — The `Makefile` committed in this repository is the original,
> **non-compliant** version of the file (see
> [Initial state](#initial-state-the-non-compliant-makefile)). It is kept on purpose to
> document the failing starting point; the corrected version is shown in the
> [Solution](#solution) section.

## Task

The end state must satisfy the following:

- The Makefile declares six targets, with these behaviours:
  - `setup` — creates a virtual environment at `mlops-venv/` and installs dependencies from `requirements.txt`;
  - `data` — runs `python3 src/data/process_data.py`;
  - `train` — runs `python3 src/models/train.py`;
  - `test` — runs `pytest tests/`;
  - `clean` — recursively removes every `__pycache__` directory, removes `.pytest_cache`, and clears the contents of `models/`;
  - `all` — runs `setup`, `data`, `train`, and `test`, in that order.
- All six target names are declared as `.PHONY`, so that Make never confuses them with
  files of the same name.
- `make all` completes without error.
- Recipe lines are indented with a real **tab** character, not spaces.

## Initial state: the non-compliant Makefile

Running the file as committed:

```console
$ make all
Makefile:7: *** missing separator.  Stop.
```

The offending line is the `data` recipe, which is indented with four **spaces** instead
of a tab:

```make
data:
    python3 src/data/process_data.py   # <-- 4 spaces: make cannot parse this
```

`make` accepts only three kinds of lines: a target/rule (`name:` starting at column 1), a
variable assignment, or a recipe line that begins with a **tab**. A space-indented line is
none of those, so parsing aborts with `missing separator` and reports the first line it
could not interpret.

Beyond the indentation, the file was also incomplete:

| Requirement | Initial state |
| --- | --- |
| `all` runs `setup`, `data`, `train`, `test` | `all: setup train test` — `data` missing |
| `clean` removes every `__pycache__` recursively | `rm -rf __pycache__` — not recursive |
| `clean` removes `.pytest_cache` | missing |
| `clean` clears the contents of `models/` | missing |
| `.PHONY` for all six targets | missing |

## Solution

```make
# fraud-detection Makefile

.PHONY: setup data train test clean all

setup:
	python3 -m venv mlops-venv && mlops-venv/bin/pip install -r requirements.txt

data:
	python3 src/data/process_data.py

train:
	python3 src/models/train.py

test:
	pytest tests/

clean:
	find . -type d -name "__pycache__" -exec rm -rf {} +
	rm -rf .pytest_cache
	rm -rf models/*

all: setup data train test
```

Two changes are essential:

1. **Tab indentation.** The `data` recipe now starts with a literal tab, which is what
   resolves the `missing separator` error. A quick way to normalise an existing file is
   `sed -i 's/^    /\t/' Makefile`, followed by `cat -A Makefile` to confirm that every
   recipe line starts with `^I`.
2. **Completing the targets.** `all` now includes `data`, `clean` performs all three
   cleanup actions, and every target is declared `.PHONY`.

## What `.PHONY` does

By default, `make` treats every target as a *file*: before running a recipe it checks
whether a file with that name exists and whether it is older than its prerequisites, and
if the file looks up to date it skips the recipe silently. For a task runner like this one
the targets are *actions*, not build artifacts, so that assumption is both wrong and
dangerous — if a file or directory named `clean`, `data` or `test` happens to exist in the
project root, `make clean` would print `make: 'clean' is up to date.` and do nothing at
all, which is a silent failure rather than an error. Declaring
`.PHONY: setup data train test clean all` tells `make` that those names are not files, so
it ignores the filesystem entirely and always executes the recipes; it additionally
prevents the built-in implicit rules from ever matching those names. In short, `.PHONY`
makes the intent explicit and turns a potential silent no-op into a guaranteed execution.

## Cross-check

Run the following from the project directory to verify the end state.

**1. Recipe lines are tab-indented**

```bash
cat -A Makefile          # every recipe line must start with ^I, never spaces
```

**2. `.PHONY` declares all six targets**

```bash
grep -n '^\.PHONY' Makefile

# and prove it behaves as expected:
touch clean
make clean               # "make: 'clean' is up to date." means the target is NOT phony
rm -f clean
```

**3. `clean` performs all three cleanup actions**

```bash
mkdir -p src/__pycache__ tests/__pycache__ .pytest_cache
touch src/__pycache__/a.pyc tests/__pycache__/b.pyc .pytest_cache/README.md
touch models/dummy.txt

make clean

find . -type d -name '__pycache__'   # expected: no output
[ -d .pytest_cache ] && echo "STILL PRESENT" || echo ".pytest_cache removed"
ls -A models/                        # expected: no output
```

**4. `make all` runs every stage in order and exits cleanly**

```bash
make all; echo "exit code = $?"
```

Expected: `setup` → `data` → `train` → `test` (with `1 passed`) and `exit code = 0`.

> `__pycache__` and `.pytest_cache` are recreated by every test run, so run the `clean`
> checks *after* `make clean`, not after `make all`.

## Repository notes

- The `Makefile` in this directory is the non-compliant version that produced the failure;
  the corrected file is documented in the [Solution](#solution) section above.
