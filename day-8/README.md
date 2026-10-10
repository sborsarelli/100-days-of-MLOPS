# Day 8 — Fixing the `pre-commit` Configuration for the Fraud-Detection Repo

## Overview

Part of the *100 Days of MLOps* challenge. This exercise repairs a broken
`.pre-commit-config.yaml` so that the git hook suite runs cleanly on every commit,
documenting what `pre-commit` is, the failing starting point, the fix, and the commands
used to verify the end state.

The scenario: the xFusionCorp Industries ML team enforces code quality on **every commit**
with [`pre-commit`](https://pre-commit.com/). A draft `.pre-commit-config.yaml` lives in
the git repository at `/root/code/fraud-detection/` (alongside `process.py`), but it does
not match the team's standards, so `pre-commit run --all-files` fails. The configuration
must be revised until it complies.

> **Note** — The configuration shown in
> [Initial state](#initial-state-the-non-compliant-config) is the original,
> **non-compliant** version, kept on purpose to document the failing starting point. The
> corrected version is shown in the [Solution](#solution) section.

## Task

The end state must satisfy the following:

- The configuration declares these **five hooks**, so that `pre-commit run --all-files`
  executes every one of them:
  - `trailing-whitespace`, `end-of-file-fixer`, and `check-yaml` — all three sourced from
    the `pre-commit/pre-commit-hooks` repository, pinned to a current release;
  - `ruff` — sourced from the `astral-sh/ruff-pre-commit` repository, pinned to a current
    release;
  - `black` — sourced from the `psf/black-pre-commit-mirror` repository, pinned to a
    current release.
- **Every** repository entry in the configuration includes a `rev:` field.
- The hooks are **registered with git** and **run cleanly** against the tracked files.

> **Tip** — `pre-commit autoupdate` queries each referenced repository and rewrites the
> `rev:` pins to the latest released tag. This is the standard way to discover current
> versions without looking them up by hand.

## Background: what `pre-commit` is

`pre-commit` is a **framework for managing and running git hooks**. A git hook is a script
that git runs at a specific point in its lifecycle; the `pre-commit` hook runs just before
a commit is finalised, which makes it the ideal place to catch formatting and linting
problems **before** they enter the history.

The framework is not a linter or a formatter itself — it is the *manager* that downloads,
isolates, and runs other tools. Its advantages over hand-written `.git/hooks` scripts:

- A single declarative config file: **`.pre-commit-config.yaml`**.
- Each hook runs in its **own isolated environment**, so hook dependencies never clash
  with the project's dependencies.
- Versions are **pinned** in the config, so every developer runs exactly the same tool
  versions.
- A huge ecosystem of reusable hooks (formatters, linters, secret scanners, …).

Minimal structure:

```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks   # where the hook lives
    rev: v6.0.0                                            # pinned revision (tag/SHA)
    hooks:
      - id: trailing-whitespace                            # which hook(s) to run
```

| Key | Meaning |
| --- | --- |
| `repo` | URL of the git repository that defines the hooks |
| `rev` | revision to use: a **tag** or commit SHA (branch/moving tags are unsupported) |
| `hooks` | list of hooks to run from that repository; each needs an `id` |

Common commands:

| Command | Effect |
| --- | --- |
| `pre-commit install` | installs the hook into `.git/hooks/pre-commit` (registers it with git) |
| `pre-commit run --all-files` | runs the hooks against every tracked file |
| `pre-commit run` | runs the hooks against the staged files |
| `pre-commit autoupdate` | rewrites each `rev:` to the latest release |
| `pre-commit clean` | clears the cached hook environments |

One behaviour worth internalising early: several hooks **auto-fix** files
(`trailing-whitespace`, `end-of-file-fixer`, `black`). When one fixes a file it still
exits with an error, so the first run reports `Failed` — you re-stage the fixed files and
run again, and the second run passes. This is normal, not a misconfiguration.

## Initial state: the non-compliant config

```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v2.3.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check_yaml

  - repo: https://github.com/charliermarsh/ruff-pre-commit
    rev: v0.1.0
    hooks:
      - id: ruff-lint

  - repo: https://github.com/psf/black-pre-commit-mirror
    hooks:
      - id: black
```

Running `pre-commit run --all-files` fails immediately:

```console
$ pre-commit run --all-files
An error has occurred: InvalidConfigError:
==> File .pre-commit-config.yaml
==> At Config()
==> At key: repos
==> At Repository(repo='https://github.com/psf/black-pre-commit-mirror')
=====> Missing required key: rev
```

| Requirement | Initial state |
| --- | --- |
| `check-yaml` from `pre-commit/pre-commit-hooks` | `id: check_yaml` (underscore instead of hyphen) |
| `ruff` from `astral-sh/ruff-pre-commit` | `repo: .../charliermarsh/ruff-pre-commit` (old organisation name) |
| `ruff` hook id | `id: ruff-lint` (not a valid hook id) |
| every repo entry has `rev:` | the `black` entry has **no** `rev:` → `Missing required key: rev` |
| hooks pinned to a *current* release | `v2.3.0` and `v0.1.0` are old releases |

The `rev` error is the one that aborts the run, but all five issues must be fixed for the
configuration to meet the team's standards.

## Solution

### 1. Correct the repository URLs and hook ids

`autoupdate` only rewrites `rev:` values — it does **not** repair repository URLs or hook
ids. Fix those first:

- `charliermarsh/ruff-pre-commit` → `astral-sh/ruff-pre-commit` (the project moved orgs);
- `check_yaml` → `check-yaml`;
- `ruff-lint` → `ruff`;
- add the missing `rev:` to the black entry (any valid ref, so `autoupdate` can resolve it).

### 2. Pin the current releases with `pre-commit autoupdate`

```console
$ pre-commit autoupdate
[https://github.com/pre-commit/pre-commit-hooks] updating v2.3.0 -> v6.0.0
[https://github.com/astral-sh/ruff-pre-commit] updating current -> v0.16.10
[https://github.com/psf/black-pre-commit-mirror] updating current -> 26.10.0
```

### 3. The corrected `.pre-commit-config.yaml`

```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v6.0.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml

  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.16.10
    hooks:
      - id: ruff

  - repo: https://github.com/psf/black-pre-commit-mirror
    rev: 26.10.0
    hooks:
      - id: black
```

> The exact `rev` values are whatever `pre-commit autoupdate` resolved to at the time of
> the exercise — they are not hard-coded requirements.

### 4. Run the hooks and let them fix the tracked files

```console
$ pre-commit run --all-files
trim trailing whitespace.................................................Failed
- hook id: trailing-whitespace
- files were modified by this hook

Fixing process.py

fix end of files.........................................................Passed
check yaml...............................................................Passed
ruff (legacy alias)......................................................Passed
black....................................................................Passed
```

`trailing-whitespace` **auto-fixed** `process.py` (removed trailing spaces), so this run
reports `Failed`. Re-run; now `end-of-file-fixer` adds the missing final newline and also
reports `Failed`; re-run a third time and everything passes:

```console
$ pre-commit run --all-files
trim trailing whitespace.................................................Passed
fix end of files.........................................................Passed
check yaml...............................................................Passed
ruff (legacy alias)......................................................Passed
black....................................................................Passed
```

### 5. Register the hook with git

```console
$ pre-commit install
pre-commit installed at .git/hooks/pre-commit
```

### 6. Commit

```bash
git add process.py .pre-commit-config.yaml
git commit -m "fix: pre-commit config and clean process.py"
```

The hook now runs on the commit itself and passes:

```console
$ git commit -m "fix: pre-commit config and clean process.py"
trim trailing whitespace.................................................Passed
fix end of files.........................................................Passed
check yaml...............................................................Passed
ruff (legacy alias)......................................................Passed
black....................................................................Passed
[main f1aed4c] fix: pre-commit config and clean process.py
 2 files changed, 7 insertions(+), 6 deletions(-)
```

## Verification

The end state is reached when, from `/root/code/fraud-detection`:

1. `pre-commit run --all-files` prints **`Passed`** for all five hooks;
2. `.git/hooks/pre-commit` exists (the hook is registered with git);
3. a `git commit` triggers the hooks and succeeds.

## Gotchas

### Underscore vs hyphen in hook ids

The `pre-commit-hooks` ids use **hyphens**: `check-yaml`, not `check_yaml`. A wrong id is
not silently ignored — pre-commit errors out when it cannot find the hook in the repo.

### `autoupdate` fixes versions, not URLs or ids

`pre-commit autoupdate` only rewrites `rev:`. If the repository URL is stale (here,
`charliermarsh/ruff-pre-commit`) or the hook id is wrong (`ruff-lint`), `autoupdate` will
not correct it — those must be edited by hand.

### Mutable references are rejected

A `rev:` pointing at a branch or a moving tag produces
`appears to be a mutable reference (moving tag / branch)`. Pin an immutable tag or SHA;
`autoupdate` often resolves this for you.

### The config file is itself checked by the hooks

`.pre-commit-config.yaml` is a tracked file, so the hooks inspect it too. A stray **trailing
space** left after a `rev:` value (e.g. `rev: v0.16.10␠`) makes the `trailing-whitespace`
hook fail on the config file itself — so the run never comes back clean. Keep the config
tidy.

### A `Failed` on the first run can simply mean "auto-fixed"

`trailing-whitespace`, `end-of-file-fixer` and `black` modify files and then exit non-zero.
Re-stage the modified files and re-run; the second pass should be green.

### `ruff (legacy alias)`

The `ruff` hook id is still exposed as a legacy alias by current `ruff-pre-commit`
releases. It works, but the modern split is `ruff-check` (linting) and `ruff-format`
(formatting). Not required by this task.

## Further reading

### Book

- **Publishing Python Packages** — Dane Hillard (Manning, 2022, ISBN 9781617299919).
  Chapter 9, *"Reducing rework using pre-commit hooks"*, is a dedicated, beginner-friendly
  walkthrough that sets up pre-commit alongside `black`, `mypy`, `flake8`, `tox` and a
  GitHub Actions pipeline.
  <https://www.manning.com/books/publishing-python-packages>
  - *Note:* there is no book devoted **exclusively** to pre-commit; this is the closest
    single source with a full chapter on the subject. For the wider "code-quality safety
    net" context (linters, static analysis, testing), a good companion is
    **Robust Python** — Patrick Viafore (O'Reilly, 2021).

### Online resources

- Official documentation — <https://pre-commit.com/> — the reference for every
  configuration key and command.
- Supported hooks index — <https://pre-commit.com/hooks.html> — searchable catalogue of
  the hooks referenced in this exercise.
- Framework source — <https://github.com/pre-commit/pre-commit>.
- `pre-commit-hooks` repository — <https://github.com/pre-commit/pre-commit-hooks> — the
  definitions of `trailing-whitespace`, `end-of-file-fixer`, `check-yaml`, and more.
- Stefanie Molin, *"How to Set Up Pre-Commit Hooks"* —
  <https://stefaniemolin.com/articles/devx/pre-commit/setup-guide/> — a step-by-step,
  beginner-oriented setup guide with a companion example repository.

### Video tutorials

- **PyBites — "Increase Python code quality with pre-commit"** —
  <https://www.youtube.com/watch?v=XFyLzr5Ehf0> — the best starting point for a junior
  developer: short, practical, and it walks through installing pre-commit and wiring up
  `black`, `isort` and friends end to end.
- *"Improve your Git Commit Quality with Pre-Commit Hooks"* —
  <https://www.youtube.com/watch?v=U-zTgdRmLms> — a longer, live-demo session that goes
  from zero to a working configuration, including debugging a failing hook.
- GitGuardian — *"Complete guide to GitHooks"* —
  <https://www.youtube.com/watch?v=ObksvAZyWdo> — broader context on how git hooks work
  under the hood, useful for understanding what `pre-commit install` actually writes.

## Repository notes

- This directory documents the exercise; the project itself lives at
  `/root/code/fraud-detection/` on the lab sandbox and is not committed here.
- `task.txt` is the exercise brief and is kept local only (see `.gitignore`).
- `pre-commit.md` is a companion tutorial on pre-commit for reference.
