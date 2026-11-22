# Day 17 — Running and Promoting DVC Experiments

## Overview

Part of the *100 Days of MLOps* challenge. Day 16 registered `metrics.json` as a
DVC metric; this day uses **DVC experiments** to sweep a hyperparameter, compare
the tracked runs by `f1_score`, and promote the winner into the workspace — all
without ever touching `params.yaml` by hand or losing a `metrics.json`.

The `fraud-detection` pipeline at `/root/code/fraud-detection/` is parameterised
by `max_depth` (baseline `4`, shallow enough to underfit). Three experiments are
run over `2, 6, 12`, the `f1_score` column is compared, and the best run is
applied so its params, metrics and `models/model.pkl` become the tracked
workspace state.

## Task

The environment:

| Fact | Value |
| --- | --- |
| Project | `/root/code/fraud-detection/` (DVC initialised, 3-stage pipeline) |
| Stages | `process_data` → `split_data` → `train` |
| Params | `params.yaml`: `n_estimators: 100`, `max_depth: 4` |
| Metric | `train` stage declares `metrics.json` under `metrics:` with `cache: false` |
| Constraint | Do not modify the Python files |

Acceptance criteria:

- Three DVC experiments run, each with a different `max_depth` (e.g. 2, 6, 12);
  each retrains the model and produces a fresh `metrics.json`.
- The experiment with the highest `f1_score` is applied to the workspace, so its
  `max_depth`, `metrics.json`, and `models/model.pkl` become the tracked state.
- The DVC extension's **EXPERIMENTS** view lists every experiment alongside its
  parameters and metrics.

## Background: how `dvc exp` stores results

### Experiments run in a temporary workspace copy

When queued with `--queue` and executed with `--run-all`, each experiment runs in
an isolated workspace copy (by default under `.dvc/tmp/exps` or an equivalent
temp directory, depending on DVC version), so `params.yaml`, `metrics.json` and
`models/model.pkl` in the real working tree are not touched. Each run's params,
metrics and outputs are saved in an internal git ref (`refs/exps/...`) plus the
DVC cache — nothing is overwritten or lost between queued runs.

A non-queued `dvc exp run` may modify the working tree directly unless you pass
`--temp`. The `--queue` / `--run-all` pattern used here is the safest way to keep
the workspace unchanged until you explicitly apply the winner.

The workspace files only change when the chosen experiment is *promoted* with
`dvc exp apply`, which restores that run's params, metrics and outputs (the
`model.pkl` comes back from the cache; `metrics.json` from the experiment ref,
since it is `cache: false`).

### Parameter sweeps with `-S`

`--set-param` (`-S`) uses Hydra-style override syntax:

- list of values: `-S 'max_depth=2,6,12'` → queues one experiment per value
- range: `-S 'max_depth=range(2,13,5)'`

A single command therefore produces *n* experiments — no loop, no manual
`params.yaml` edits. With `--queue` the runs are only scheduled; `dvc exp run
--run-all` executes them sequentially. Without `--queue`, each value may run
directly in the workspace and modify files, so the queued pattern is the safer
choice.

A sweep uses auto-generated names such as `lucky-curb`, `lobar-vlei`, etc.
Use `dvc exp run --queue --name <name> -S 'max_depth=<value>'` to name a single
run, or rename experiments afterwards with `dvc exp rename <old> <new>`.

### `--drop` vs `--keep` in `dvc exp show`

Counterintuitive: `--keep <regex>` does **not** select columns — it *protects*
matching columns from removal by other options. To hide columns use
`--drop <regex>` on the column names (dep columns are named after their file
paths, e.g. `data/...`, `src/...`).

## Solution

Run from `/root/code/fraud-detection`.

### 1. Queue and run the sweep

```bash
dvc exp run --queue -S 'max_depth=2,6,12'
dvc exp run --run-all
```

Each run reports its metrics in the logs:

```console
max_depth=2,  metrics={'accuracy': 0.6,   'f1_score': 0.2453}
max_depth=6,  metrics={'accuracy': 0.745, 'f1_score': 0.6483}
max_depth=12, metrics={'accuracy': 0.75,  'f1_score': 0.6795}
```

### 2. Compare experiments

```bash
# readable table: hide dep-hash, state and constant-param columns
dvc exp show --drop 'data/|src/|n_estimators|Created|State|Executor'
```

```console
 ────────────────────────────────────────────────────────────
  Experiment                 accuracy   f1_score   max_depth
 ────────────────────────────────────────────────────────────
  workspace                      0.75     0.6795   12
  main                           0.71     0.5735   4
  ├── 7948722 [ready-lats]       0.75     0.6795   12
  ├── 0422dab [lobar-vlei]      0.745     0.6483   6
  └── a1207eb [lucky-curb]        0.6     0.2453   2
 ────────────────────────────────────────────────────────────
```

### 3. Promote the winner

`ready-lats` (max_depth=12, f1=0.6795) wins over the baseline (0.5735):

```bash
dvc exp apply ready-lats
```

The workspace now holds that run's `params.yaml` (`max_depth: 12`),
`metrics.json` and `models/model.pkl` — the tracked state required by the task.
Commit `params.yaml`, `dvc.lock` and `metrics.json` to pin it in version control.

## Verification

```console
$ cat params.yaml
n_estimators: 100
max_depth: 12

$ cat metrics.json
{
  "accuracy": 0.75,
  "f1_score": 0.6795
}
```

| Experiment | max_depth | f1_score | outcome |
| --- | --- | --- | --- |
| lucky-curb | 2 | 0.2453 | Success |
| lobar-vlei | 6 | 0.6483 | Success |
| ready-lats | 12 | **0.6795** | Success → applied to workspace |

## Gotchas

### `Key 'train' is not in struct` — stage prefix vs flat params

`-S 'train.max_depth=2'` fails on a flat `params.yaml`:

```console
ERROR: Invalid `--set-param` value: Could not override 'train.max_depth'.
To append to your config use +train.max_depth=2: Key 'train' is not in struct
```

The Hydra override is resolved against the params file contents, whose top-level
keys are `n_estimators` and `max_depth` — there is no `train` section. Use the
bare key (`-S 'max_depth=2'`) or qualify the file explicitly
(`-S 'params.yaml:max_depth=2'`). The `stage.param` form only fits nested
params files.

### `WARNING: Unable to find less`

`dvc exp show` pipes into `less --chop-long-lines` when available; without it the
table is printed raw — harmless. Use `--no-pager`, install `less`, or set
`DVC_PAGER=more`.

### `--keep` does not mean "show only"

`dvc exp show --keep 'f1_score'` leaves every column in place: `--keep` marks
columns that must survive `--drop`/`--only-changed`, it never hides the rest.
Filtering is always done with `--drop`.

## Command cheat sheet

```bash
dvc exp run --queue -S 'max_depth=2,6,12'        # queue a param sweep
dvc exp run --run-all                            # execute the queue sequentially
dvc exp show --drop 'data/|src/'                 # compare, hide dep hashes
dvc exp show --only-changed                      # hide columns equal across runs
dvc exp apply <exp-name>                         # promote winner to workspace
dvc exp rename <old> <new>                       # rename an experiment
dvc exp remove --queue                           # clear a dirty queue
```

## Further reading

- `dvc exp run` — <https://doc.dvc.org/command-reference/exp/run> — `-S`
  override syntax, `--queue`, `--run-all`.
- `dvc exp show` — <https://doc.dvc.org/command-reference/exp/show> —
  `--drop`/`--keep`/`--only-changed` semantics and output formats.
- `dvc exp apply` — <https://doc.dvc.org/command-reference/exp/apply> —
  restoring an experiment into the workspace.
- **Running Experiments** —
  <https://doc.dvc.org/user-guide/experiment-management/running-experiments> —
  how experiments execute in temp copies and where they are stored.
