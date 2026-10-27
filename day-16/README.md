# Day 16 — Exposing Model Metrics with `dvc metrics`

## Overview

Part of the *100 Days of MLOps* challenge. Days 14–15 built the pipeline and wired
hyperparameters into it; this day surfaces the **results**: registering
`metrics.json` so DVC can report and compare model performance.

The `fraud-detection` pipeline at `/root/code/fraud-detection/` already trains a
model and writes `metrics.json` — but the file is declared as a regular `outs`
entry, so DVC caches it like any other artifact and does not recognise it as a
metric. `dvc metrics show` therefore reports nothing. The fix is to re-declare
the file under the `metrics:` key with `cache: false`, so the small JSON lives in
Git (for diff history) instead of the DVC cache.

## Task

The environment:

| Fact | Value |
| --- | --- |
| Project | `/root/code/fraud-detection/` (DVC initialised, 3-stage pipeline) |
| Stages | `process_data` → `split_data` → `train` |
| Script | `src/models/train.py` writes `models/model.pkl` and `metrics.json` |
| Constraint | Do not modify the Python files |

Acceptance criteria:

- The `train` stage in `dvc.yaml` declares `metrics.json` as a DVC **metric**
  output rather than a regular file output, with `cache: false` so the JSON lives
  in Git for diff history rather than in the DVC cache.
- The pipeline has been reproduced so the metric registration takes effect, and
  `dvc metrics show` reports the `accuracy` and `f1_score` values from
  `metrics.json`.
- Tip: once registered, `dvc metrics diff` compares metric values across Git
  commits — useful when iterating on the model.

## Background: `outs` vs `metrics`

### Same file, different role

A stage output can be declared under three different keys, each giving DVC a
different piece of information about the file:

| Key | Meaning | Typical contents |
| --- | --- | --- |
| `outs` | generic artifact, hashed and pushed to the DVC cache | models, datasets, binaries |
| `metrics` | structured key/value file comparable across runs | `accuracy`, `f1_score`, AUC |
| `plots` | data series file for visual comparison | ROC/PR curves, confusion matrices |

A file under `outs` is opaque to DVC: it is stored and versioned, but its
contents are never interpreted. A file under `metrics` must be a structured file
(JSON, YAML or TOML) mapping metric names to numbers — and only then do
`dvc metrics show` and `dvc metrics diff` pick it up.

### Why `cache: false`

Metrics files are tiny text files. Keeping them in the DVC cache would hide them
from `git diff` and from GitHub's UI. With `cache: false`:

- DVC still records the file in `dvc.lock` and knows it is a stage output;
- the file itself is committed to Git like source code;
- `dvc metrics diff` can then compare the value at `HEAD` against the workspace —
  the whole point of tracking metrics.

One side effect to know: if *any* output of a stage has `cache: false`, DVC
deactivates the **run cache** for that stage (it cannot restore a run purely
from cache). That is an acceptable trade-off for metrics.

### No new stage needed

A first intuition might be "add an `evaluate` stage that produces the metrics" —
unnecessary here: `train.py` already writes `metrics.json`. The only change is
*moving the declaration* from `outs` to `metrics` inside the existing `train`
stage.

## Solution

Run from `/root/code/fraud-detection`.

### 1. Re-declare `metrics.json` in `dvc.yaml`

Edit the `train` stage — remove `metrics.json` from `outs` and add it under
`metrics`:

```yaml
  train:
    cmd: python3 src/models/train.py
    deps:
      - data/processed/train.csv
      - src/models/train.py
    outs:
      - models/model.pkl
    metrics:
      - metrics.json:
          cache: false
```

(The equivalent CLI flags are `dvc stage add -M metrics.json` /
`--metrics-no-cache`, but a direct edit is simpler for an existing stage.)

### 2. Reproduce and inspect

```bash
dvc repro
dvc metrics show
```

`dvc repro` regenerates `dvc.lock` with `metrics.json` now recorded under the
`metrics` section of the `train` stage.

### 3. Commit for diff history

Because the file is no longer DVC-cached, it must be tracked by Git:

```bash
git add metrics.json dvc.yaml dvc.lock
git commit -m "Register metrics.json as a DVC metric"
```

## Verification

**1. Validation passes and the pipeline reproduces**

```console
$ dvc repro
Running stage 'process_data':
> python3 src/data/process_data.py
Running stage 'split_data':
> python3 src/data/split_data.py
Running stage 'train':
> python3 src/models/train.py
Metrics: {'accuracy': 1.0, 'f1_score': 1.0}
Updating lock file 'dvc.lock'
```

**2. Metrics are surfaced**

```console
$ dvc metrics show
Path          accuracy    f1_score
metrics.json  1.0         1.0
```

## Gotchas

### `cache: fasle` — YAML typos fail schema validation

`dvc.yaml` is validated against a strict schema at every command. A typo in a
boolean produces a precise error naming the file, path, line and column:

```console
$ dvc repro
'./dvc.yaml' validation failed.

expected bool, in stages -> train -> metrics -> 0 -> metrics.json -> cache,
line 28, column 11
```

The message tells you exactly where to look — it is a typo (`fasle`), not an
indentation problem.

### The `metrics:` list item is a mapping

Note the shape of the entry — a list item whose single key is the path and whose
value holds the output subfields:

```yaml
metrics:
  - metrics.json:        # list item
      cache: false       # subfield of that item, indented deeper
```

For a metric without options, the bare form `metrics: [metrics.json]` also
works; the mapping form is required once you add `cache: false` (or `persist`,
`remote`, `desc`, `push` — the same subfields `outs` accepts).

### `metrics.json` disappears from Git status handling

With `cache: false`, DVC no longer writes the file into `models/.gitignore`-style
ignores, so `metrics.json` shows up as an untracked/changed file in `git status`.
That is expected — commit it.

## Further reading

- **`dvc.yaml` file format** —
  <https://doc.dvc.org/user-guide/project-structure/dvcyaml-files> — the canonical
  reference for the `metrics` key and the `cache` output subfield. This is the
  page that documents the manual syntax.
- `dvc metrics` — <https://doc.dvc.org/command-reference/metrics> — `show`,
  `diff`, supported file formats (JSON/TOML/YAML) and how files are discovered.
- `dvc stage add` — <https://doc.dvc.org/command-reference/stage/add> — the `-m` /
  `-M` flags, i.e. the CLI equivalent of `metrics:` + `cache: false`.
- **Defining Pipelines** —
  <https://doc.dvc.org/user-guide/pipelines/defining-pipelines> — where metrics
  and plots fit among stage outputs.
- **Get Started: Metrics, Parameters, and Plots** —
  <https://doc.dvc.org/start/data-pipelines/metrics-parameters-plots> — the
  tutorial flow, DVCLive-oriented; see `tutorial.md` in this directory for how it
  relates to the manual `dvc.yaml` syntax.
