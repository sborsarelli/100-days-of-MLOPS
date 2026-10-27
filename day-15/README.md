# Day 15 — Tracking Hyperparameters with DVC Params

## Overview

Part of the *100 Days of MLOps* challenge. Day 14 turned loose scripts into a
reproducible pipeline declared in `dvc.yaml`; this day tackles the configuration
side: making DVC **aware of a hyperparameter** so that changing it automatically
triggers retraining.

The xFusionCorp Industries ML team manages model hyperparameters in `params.yaml`.
In the `fraud-detection` project at `/root/code/fraud-detection/`, the `train`
stage already reads `n_estimators` from that file — but the parameter is not
declared to DVC, so editing its value would leave the pipeline "up to date" and
silently keep the stale model. The goal is to declare the parameter in `dvc.yaml`
and demonstrate parameter-driven reproducibility.

## Task

The environment:

| Fact | Value |
| --- | --- |
| Project | `/root/code/fraud-detection/` (DVC initialised, 3-stage pipeline) |
| Stages | `process_data` → `split_data` → `train` |
| Params file | `params.yaml` declaring `n_estimators: 100` |
| Script | `src/models/train.py` reads `n_estimators` from `params.yaml` |
| Constraint | Do not modify the Python files |

Acceptance criteria:

- The `train` stage in `dvc.yaml` lists `n_estimators` under a `params:` section,
  and the pipeline has been reproduced.
- Parameter-driven retraining is demonstrated: changing `n_estimators` (e.g. to
  200) and re-running the pipeline re-executes **only** the `train` stage,
  records the new value in `dvc.lock`, and regenerates `models/model.pkl`.
- `dvc params diff` reports changes to the tracked parameter values across Git
  commits — useful when comparing experiments.

## Background: how DVC params work

### `deps` track files, `params` track values

A file listed under `deps` is hashed in full. `params.yaml` would work as a plain
dependency, but that couples the stage to the *whole file*: any unrelated edit
(e.g. adding a comment or a second parameter) would mark `train` stale.

Declaring `params:` instead makes DVC track the **individual values** inside the
file (YAML, JSON, TOML or Python format). The stage is re-run only when one of
the listed keys actually changes.

### Where the values live

- `dvc.yaml` declares *which* keys to watch:

  ```yaml
  train:
    cmd: python3 src/models/train.py
    deps:
      - data/processed/train.csv
      - src/models/train.py
    params:
      - n_estimators          # looked up in params.yaml by default
    outs:
      - models/model.pkl
  ```

- `dvc.lock` records the value seen at run time:

  ```yaml
  train:
    params:
      params.yaml:
        n_estimators: 200
  ```

Committing `params.yaml` + `dvc.lock` to Git is what makes
`dvc params diff` able to compare experiments across commits.

## Solution

Run from `/root/code/fraud-detection` (the DVC project root — the scripts use
relative paths, so the working directory matters).

### 1. Declare the parameter on the existing stage

Either edit `dvc.yaml` by hand and add `params:` under `train`, or regenerate the
stage declaration with `dvc stage add`:

```bash
dvc stage add --force -n train \
  -p n_estimators \
  -d src/models/train.py \
  -d data/processed/train.csv \
  -o models/model.pkl \
  python3 src/models/train.py
```

- `-p n_estimators` registers the key from `params.yaml` (`-p` defaults to that
  file; use `-p other.yaml:key` for a different file).
- `--force` is required because stage `train` already exists and must be
  overwritten.

### 2. Reproduce

```bash
dvc repro
```

All three stages run once; `dvc.lock` now contains the
`params: {params.yaml: {n_estimators: 100}}` block under `train`.

### 3. Demonstrate parameter-driven retraining

Edit `params.yaml` (`n_estimators: 100` → `200`), then:

```bash
dvc repro
dvc params diff
```

Only `train` re-executes; `process_data` and `split_data` are skipped as
unchanged.

## Resulting `dvc.yaml` (`train` stage)

```yaml
  train:
    cmd: python3 src/models/train.py
    deps:
    - data/processed/train.csv
    - src/models/train.py
    outs:
    - models/model.pkl
    params:
    - n_estimators
```

## Verification

**1. Parameter recorded in the lock file**

```console
$ grep -A3 'params' dvc.lock
    params:
      params.yaml:
        n_estimators: 100
```

**2. Only `train` re-runs after the change**

```console
$ dvc repro
Stage 'process_data' didn't change, skipping
Stage 'split_data' didn't change, skipping
Running stage 'train':
> python3 src/models/train.py
Trained RandomForestClassifier with n_estimators=200
Updating lock file 'dvc.lock'
```

**3. Diff across commits**

```console
$ dvc params diff
Path         Param         HEAD    workspace
params.yaml  n_estimators  100     200
```

## Gotchas

### `--force` must come before the command

Everything placed **after** the stage command is swallowed into `cmd` itself.
This silently fails to force:

```console
$ dvc stage add -n train -p n_estimators -d ... -o models/model.pkl \
    python3 src/models/train.py --force
ERROR: Stage 'train' already exists in 'dvc.yaml'. Use '--force' to overwrite.
```

Here `--force` became part of the recorded command, not a DVC flag. Put it right
after `stage add` (or before the command), as in the solution above.

### Params are opt-in, not automatic

DVC never inspects the script to discover that `train.py` opens `params.yaml` —
just like it never parses the command line for dependencies (Day 14). If the
file is read at runtime but not declared, the pipeline is broken by
construction: it *runs*, but it is not *reproducible*.

### Commit the params file itself

`dvc params diff` compares the workspace against Git commits, so `params.yaml`
must be tracked by Git (it is a small text file — do not `dvc add` it):

```bash
git add params.yaml dvc.yaml dvc.lock
```

## Further reading

- **Get Started: Metrics, Parameters, and Plots** —
  <https://doc.dvc.org/start/data-pipelines/metrics-parameters-plots> — the
  official tutorial covering `params.yaml`, `dvc params diff` and the
  DVCLive-assisted workflow.
- `dvc params` — <https://doc.dvc.org/command-reference/params> — how parameter
  tracking and `params diff` work.
- `dvc stage add` — <https://doc.dvc.org/command-reference/stage/add> — the `-p` /
  `--params` flag and `--force` semantics.
- `dvc.yaml` file format —
  <https://doc.dvc.org/user-guide/project-structure/dvcyaml-files> — the `params`
  subfield, including the `file:key` syntax for non-default parameter files.
