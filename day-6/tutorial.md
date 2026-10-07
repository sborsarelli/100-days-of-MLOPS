# Tutorial — Hands-on with `ruff` and `black`

A **learning-by-doing** walkthrough. You will create a small, deliberately messy Python
file, watch both tools fail on it, and fix it step by step until both exit cleanly.

Nothing here is theoretical for long: every section tells you what to **run**, what you
**see**, and **why** it happens. By the end you will have used the exact commands that
CI runs on every pull request.

> **Tool versions used in this tutorial:** `ruff 0.16.8`, `black 26.5.1`, `Python 3.12`.
> Output may differ slightly across versions, but the concepts are identical.

---

## What you'll learn

- The difference between a **formatter** (`black`) and a **linter** (`ruff`).
- How to read `ruff`'s output and decode rule codes like `E501`, `F401`, `I001`, `W291`.
- When a problem is **auto-fixable** (`--fix`) and when it needs a human.
- How to configure both tools in `pyproject.toml` so they agree.
- How to run them as a **gate** in CI.

## Prerequisites

```bash
python3 --version     # 3.10+ recommended
ruff --version
black --version
```

If either tool is missing:

```bash
pip install ruff black
```

## The 10-minute map

| | `black` | `ruff` |
| --- | --- | --- |
| Role | **Formatter** — rewrites code into one style | **Linter** — reports problems |
| Command to check | `black --check <path>` | `ruff check <path>` |
| Command to apply | `black <path>` | `ruff check --fix <path>` |
| Exits non-zero when | a file *would* change | a rule is violated |
| Configures via | `[tool.black]` | `[tool.ruff]` + `[tool.ruff.lint]` |

The mental model to keep: **`black` decides how code looks, `ruff` decides what code is
allowed to say.** They overlap a little (both care about line length), which is why we
configure them together.

---

## Step 0 — Create a sandbox

Never experiment in a repo you care about. Make a throwaway project:

```bash
mkdir -p ~/ruffblack-tutorial/src
cd ~/ruffblack-tutorial
```

Your project now looks like:

```
~/ruffblack-tutorial/
└── src/
```

## Step 1 — Write the configuration

Create `pyproject.toml`:

```toml
[project]
name = "demo"
version = "0.1.0"

[tool.ruff]
line-length = 120

[tool.ruff.lint]
select = ["E", "F", "W", "I"]

[tool.black]
line-length = 120
```

**Why each key exists:**

| Key | Meaning |
| --- | --- |
| `[tool.ruff]` `line-length = 120` | Max line length ruff tolerates before `E501`. It is a *general* option, so it lives directly under `[tool.ruff]`. |
| `[tool.ruff.lint]` `select = [...]` | Which rule families are enabled: `E` (pycodestyle errors), `F` (pyflakes), `W` (pycodestyle warnings), `I` (import sorting / isort). This is a *lint* option, so it lives under `[tool.ruff.lint]`. |
| `[tool.black]` `line-length = 120` | How wide black is allowed to draw. Keep it **equal** to ruff's, or the two tools will fight (see [Troubleshooting](#troubleshooting)). |

> **Do this now:** save the file. You will not touch it again until
> [Step 6](#why-the-config-is-written-this-way), where we break it on purpose to see the
> errors it prevents.

## Step 2 — Meet the messy file

This file was written in a hurry. It contains at least six problems on purpose:

```bash
cat > src/pipeline.py <<'PY'
import sys
import os
import numpy as np
from pathlib import Path
import pandas as pd

DATA_PATH = Path("data/transactions.csv")
LOG_MESSAGE = "Loading transactions from the production fraud-detection data lake bucket, this message is deliberately long to exceed the limit."

def load_transactions(path=DATA_PATH,sep=','):
    return pd.read_csv(path, sep=sep)

def summarize(df ):
    total = len(df)
    columns = df.columns.tolist()
    return {"rows": total, "cols": df.shape[1], "columns": columns, "memory_bytes": int(df.memory_usage(deep=True).sum())}
PY
```

Before running anything, **guess**: which problems are about *style* (how it looks) and
which are about *correctness* (what it does)? Keep your list; you'll compare it to the
tools' reports.

---

## Step 3 — `ruff`: the linter

### 3.1 Run it and read the report

```bash
ruff check src/
```

You get (abridged — ruff prints a code snippet for each error):

```console
I001 [*] Import block is un-sorted or un-formatted
 --> src/pipeline.py:1:1
  |
1 | / import sys
2 | | import os
3 | | import numpy as np
4 | | from pathlib import Path
5 | | import pandas as pd
  | |___________________^
help: Organize imports

F401 [*] `sys` imported but unused
 --> src/pipeline.py:1:8

F401 [*] `os` imported but unused
 --> src/pipeline.py:2:8

F401 [*] `numpy` imported but unused
 --> src/pipeline.py:3:17

E501 Line too long (145 > 120)
 --> src/pipeline.py:8:121

E501 Line too long (122 > 120)
 --> src/pipeline.py:16:121

Found 6 errors.
[*] 4 fixable with the `--fix` option.
```

**See it, don't just read it:**

- The `[*]` next to a code means *"this one can be fixed automatically"*.
- `--> src/pipeline.py:8:121` is `file:line:column` — the column is where the problem
  starts (line 8, char 121 = the first character past the 120 limit).
- The summary line tells you the score: **6 errors, 4 fixable**.

### 3.2 Decode the rule codes

| Code | Family | What it means here |
| --- | --- | --- |
| `I001` | `I` — isort | The import block isn't sorted/formatted the way isort expects |
| `F401` | `F` — pyflakes | `sys`, `os`, `numpy` are imported but never used |
| `E501` | `E` — pycodestyle | Line longer than the configured 120 characters |

This is why `select = ["E", "F", "W", "I"]` matters: each letter switches on a whole family
of checks. Change the letter, change what the tool can see.

### 3.3 Auto-fix the safe ones

```bash
ruff check --fix src/
```

```console
E501 Line too long (145 > 120)
 --> src/pipeline.py:6:121

E501 Line too long (122 > 120)
 --> src/pipeline.py:14:121

Found 6 errors (4 fixed, 2 remaining).
```

Ruff rewrote the file for you: it **sorted the imports** and **removed the three unused
ones**. Look at the top of the file now:

```bash
sed -n '1,8p' src/pipeline.py
```

```python
from pathlib import Path

import pandas as pd

DATA_PATH = Path("data/transactions.csv")
LOG_MESSAGE = "Loading transactions from the production fraud-detection data lake bucket, this message is deliberately long to exceed the limit."

def load_transactions(path=DATA_PATH,sep=','):
```

`sys`, `os` and `numpy` are gone, and the imports are grouped and ordered. That is `I001`
and the three `F401`s resolved — **without you editing a single character**.

> **Note:** ruff is conservative. `--fix` only applies *safe* fixes. Anything that could
> change program behavior is left for you.

### 3.4 The leftovers: `E501`

The two `E501`s survive because a linter **cannot invent** how you want a long line
wrapped — that's a human decision. So `--fix` leaves them alone.

Don't fix them by hand yet. Run `ruff check src/` again to confirm only the two long lines
remain:

```console
E501 Line too long (145 > 120)
E501 Line too long (122 > 120)
Found 2 errors.
```

### Try it — see `W291` for yourself

The `W` family is enabled, but this file has no trailing whitespace. Create some and watch
ruff catch it:

```bash
printf 'x = 1   \n' > src/ws.py      # three trailing spaces
ruff check --select W src/ws.py
rm src/ws.py
```

```console
W291 [*] Trailing whitespace
 --> src/ws.py:1:6

Found 1 error.
[*] 1 fixable with the `--fix` option.
```

Invisible characters in your source are exactly the class of bug a linter exists to catch.

---

## Step 4 — `black`: the formatter

Ruff fixed the *logic-level* issues. But look at the code: `sep=','` has no spaces, `df )`
has a stray space, and the `return {...}` line is a monster. None of that is a *lint*
error — it's *style*. That's black's job.

### 4.1 Check (do not change anything)

```bash
black --check src/
```

```console
would reformat /home/you/ruffblack-tutorial/src/pipeline.py

Oh no! 💥 💔 💥
1 file would be reformatted.
```

`--check` is **read-only** — it's the CI-friendly form. Exit code is `1`, so a pipeline
would fail here.

### 4.2 Preview the diff

Before letting a tool rewrite your code, see what it wants to do:

```bash
black --diff src/
```

```diff
-def load_transactions(path=DATA_PATH,sep=','):
+
+def load_transactions(path=DATA_PATH, sep=","):
     return pd.read_csv(path, sep=sep)
 
-def summarize(df ):
+
+def summarize(df):
     total = len(df)
     columns = df.columns.tolist()
-    return {"rows": total, "cols": df.shape[1], "columns": columns, "memory_bytes": int(df.memory_usage(deep=True).sum())}
+    return {
+        "rows": total,
+        "cols": df.shape[1],
+        "columns": columns,
+        "memory_bytes": int(df.memory_usage(deep=True).sum()),
+    }
```

Three things to notice:

1. **Spacing:** `sep=','` → `sep=","` and `df )` → `df)`.
2. **Quotes:** black standardizes string quotes to `"` .
3. **The monster line:** black reflowed the `return {...}` into a multi-line dict — which
   also happens to fix one of the `E501`s *for free*. Formatter and linter overlap on
   line length.

### 4.3 Apply

```bash
black src/
```

```console
reformatted /home/you/ruffblack-tutorial/src/pipeline.py

All done! ✨ 🍰 ✨
1 file reformatted.
```

Run it **again** to prove it's idempotent (a formatter should never keep changing the same
file):

```bash
black src/
```

```console
All done! ✨ 🍰 ✨
1 file left unchanged.
```

---

## Step 5 — Finish the last `E501`

Black reflowed the dict, so one `E501` is gone. Confirm what's left:

```bash
ruff check src/
```

```console
E501 Line too long (145 > 120)
 --> src/pipeline.py:6:121
```

Only the long **string literal** remains. Black will not split a string for you (it would
change the string's value), and ruff won't either. This one needs a human. Wrap it with
parentheses and implicit concatenation:

```python
LOG_MESSAGE = (
    "Loading transactions from the production fraud-detection data lake bucket, "
    "this message is deliberately long to exceed the limit."
)
```

Edit `src/pipeline.py` so the `LOG_MESSAGE = "..."` line becomes the block above, then
re-check.

---

## Step 6 — Verify the end state

```bash
ruff check src/
```

```console
All checks passed!
```

```bash
black --check src/
```

```console
All done! ✨ 🍰 ✨
1 file would be left unchanged.
```

Both exit `0`. The full workflow you just performed, in order:

```bash
ruff check --fix src/     # 1. auto-fix safe lint issues
black src/                # 2. apply formatting
ruff check src/           # 3. re-check lint (formatting may reveal/absorb issues)
black --check src/        # 4. confirm the formatter is satisfied
```

The final file:

```python
from pathlib import Path

import pandas as pd

DATA_PATH = Path("data/transactions.csv")
LOG_MESSAGE = (
    "Loading transactions from the production fraud-detection data lake bucket, "
    "this message is deliberately long to exceed the limit."
)


def load_transactions(path=DATA_PATH, sep=","):
    return pd.read_csv(path, sep=sep)


def summarize(df):
    total = len(df)
    columns = df.columns.tolist()
    return {
        "rows": total,
        "cols": df.shape[1],
        "columns": columns,
        "memory_bytes": int(df.memory_usage(deep=True).sum()),
    }
```

---

## Why the config is written this way

Now break the config on purpose and watch the errors it prevents. This is the fastest way
to remember the rules.

### Mistake 1 — `select` at the top level (deprecated)

```toml
[tool.ruff]
line-length = 120
select = ["E", "F", "W", "I"]     # old location
```

```console
$ ruff check src/
warning: The top-level linter settings are deprecated in favour of their counterparts in the `lint` section. Please update the following options in `pyproject.toml`:
  - 'select' -> 'lint.select'
...
```

It still *works* (backwards compatibility), but it warns. **Fix:** move `select` under
`[tool.ruff.lint]`.

### Mistake 2 — `line-length` in the wrong section

```toml
[tool.ruff.lint]
line-length = 120                 # wrong: this is not a lint option
```

```console
$ ruff check src/
ruff failed
  Cause: Failed to parse pyproject.toml
  Cause: TOML parse error at line 6, column 1
  |
6 | line-length = 120
  | ^^^^^^^^^^^
unknown field `line-length`, expected one of `allowed-confusables`, ..., `select`, ...
```

This is a **hard failure**, not a warning. The rule of thumb:

> `line-length` → `[tool.ruff]` (general)
> `select` / `ignore` / `per-file-ignores` → `[tool.ruff.lint]` (lint-specific)

Restore the correct config from [Step 1](#step-1--write-the-configuration) before moving on.

---

## Wire it into CI

The whole point of these tools is that a machine enforces them. A minimal GitHub Actions
job:

```yaml
name: code-quality
on: [push, pull_request]

jobs:
  lint-and-format:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install ruff black
      - run: ruff check src/        # exit 1 fails the job
      - run: black --check src/     # exit 1 fails the job
```

To catch problems *before* they reach CI, add a `pre-commit` hook (`.pre-commit-config.yaml`):

```yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.16.8
    hooks:
      - id: ruff-check
        args: [--fix]
      - id: ruff-format
  - repo: https://github.com/psf/black
    rev: 26.5.1
    hooks:
      - id: black
```

Then `pip install pre-commit && pre-commit install`.

---

## Cheatsheet

| Goal | Command |
| --- | --- |
| See lint problems | `ruff check src/` |
| Auto-fix safe lint problems | `ruff check --fix src/` |
| Run one rule family only | `ruff check --select F src/` |
| Show what a rule means | `ruff rule E501` |
| Check formatting (CI mode) | `black --check src/` |
| Preview formatting changes | `black --diff src/` |
| Apply formatting | `black src/` |
| Format a single file | `black src/pipeline.py` |

## Exercises

Do these in the sandbox. Try to predict the output before running.

1. **Only unused imports.** Delete the `I` from `select` and re-run `ruff check src/` on a
   fresh messy file. What disappears?
2. **Tighten the line length.** Set both `line-length` values to `80` and re-run the
   workflow. How many new `E501`s appear?
3. **Let the tools disagree.** Set `[tool.ruff] line-length = 120` but
   `[tool.black] line-length = 88`, then run `black src/` followed by `ruff check src/`.
   What happens, and why is keeping them equal the fix?
4. **A rule you've never seen.** Run `ruff rule B006` and then write a one-line function
   that triggers it, with `select = ["B"]`.
5. **Full loop on a real file.** Pick any `.py` file you own, copy it into the sandbox, and
   run the four-command workflow. Read every diff before accepting it.

## Troubleshooting

| Symptom | Cause / Fix |
| --- | --- |
| `unknown field 'line-length'` | You put `line-length` under `[tool.ruff.lint]`. Move it to `[tool.ruff]`. |
| `warning: The top-level linter settings are deprecated` | Move `select`/`ignore` under `[tool.ruff.lint]`. |
| `black` and `ruff` keep "fighting" over lines | Their `line-length` values differ. Set them equal. |
| `E501` survives `ruff check --fix` | Long line needs a human (usually an unsplittable string). Wrap it with parentheses. |
| `black` reformats a file `ruff` just "fixed" | They have different jobs — run `ruff check --fix` **then** `black`, then `ruff check` once more. |
| `ruff: command not found` | `pip install ruff black`, or use `python -m ruff`. |
| Fixes apply to the wrong files | Pass the target explicitly (`src/`), not the whole repo. |
