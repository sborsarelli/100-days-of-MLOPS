# Day 19 — Production DVC Pipeline with SeaweedFS Remote and Release Tag

## Overview

Part of the *100 Days of MLOps* challenge. This day completes the
fraud-detection DVC pipeline end-to-end: three stages (`ingest`, `validate`,
`preprocess`) were already declared in `dvc.yaml` with one misconfiguration,
and two stages (`train`, `evaluate`) still had to be wired up. The finished
pipeline must reproduce cleanly, push its cache to a SeaweedFS remote, and be
released as a `v1.0` Git tag with every change committed.

The `ml-pipeline` project lives at `/root/code/ml-pipeline/` with Git and DVC
already initialised. `.dvc/config` is pre-configured to push to the
SeaweedFS bucket `dvc-storage` at `http://localhost:8333`.

## Task

| Fact | Value |
| --- | --- |
| Project | `/root/code/ml-pipeline/` |
| Existing stages | `ingest`, `validate`, `preprocess` |
| Missing stages | `train`, `evaluate` |
| Pre-staged scripts | `scripts-staging/train.py`, `scripts-staging/evaluate.py` |
| Params file | `params.yaml` (`n_estimators`, `max_depth`, `test_size`, `random_seed`) |
| Remote | SeaweedFS bucket `dvc-storage` |

Acceptance criteria:

- The misconfigured existing stage is corrected so `dvc repro` can complete.
- `train` is declared: depends on the preprocessed dataset and
  `scripts/train.py`; reads `n_estimators`, `max_depth`, `test_size`, and
  `random_seed` from `params.yaml`; outputs `models/model.pkl` and
  `data/processed/test_split.csv`; declares `metrics.json` as a DVC metric
  with `cache: false`.
- `evaluate` is declared: depends on `models/model.pkl`,
  `data/processed/test_split.csv`, and `scripts/evaluate.py`; outputs
  `reports/evaluation.json` declared with `cache: false`.
- The full pipeline has been reproduced, the cache pushed to the SeaweedFS
  remote, and the current state tagged `v1.0`.
- Every change is committed to Git so the release is fully captured.
- The SeaweedFS Filer bucket `/buckets/dvc-storage/` holds the pushed
  artefacts under the `files/md5/...` layout.

## Background: Git vs DVC responsibilities

Same split as Day 18, applied to a `dvc.yaml` pipeline:

- **Code and params → Git.** `scripts/*.py`, `dvc.yaml`, `params.yaml`, and
  small JSON outputs marked `cache: false` are normal Git-tracked files.
- **Data and model artefacts → DVC.** `clean.csv`, `test_split.csv`, and
  `model.pkl` are cached by DVC and pushed to the remote; Git only stores
  their hashes inside `dvc.lock`.
- **`metrics` vs `outs`.** A file declared under `metrics:` is readable via
  `dvc metrics show`; with `cache: false` it stays in the workspace and is
  committed to Git like any other file.
- **`params:` tracking.** Listing individual keys under `params:` makes DVC
  hash only those values — changing an unrelated key in `params.yaml` will
  not needlessly retrigger the stage.

## Solution

### 1. Reproduce and find the misconfigured stage

```bash
cd /root/code/ml-pipeline
dvc repro
```

`ingest` and `validate` succeed; `preprocess` fails:

```text
Running stage 'preprocess':
> python3 scripts/preprocess.py
Preprocessed: 20 clean rows
ERROR: failed to reproduce 'preprocess': output 'data/processed/cleaned.csv' does not exist
```

`scripts/preprocess.py` writes `data/processed/clean.csv`, but `dvc.yaml`
declared `data/processed/cleaned.csv`. Fix the stage output (the script was
left untouched):

```yaml
  preprocess:
    cmd: python3 scripts/preprocess.py
    deps:
      - data/raw/data.csv
      - scripts/preprocess.py
    outs:
      - data/processed/clean.csv
```

```bash
dvc repro
```

```text
Stage 'preprocess' ...
Preprocessed: 20 clean rows
Updating lock file 'dvc.lock'
```

### 2. Move the pre-staged scripts into `scripts/`

The scripts belong in `scripts/` because they are code — Git-tracked
dependencies of the pipeline, not DVC artefacts:

```bash
cp scripts-staging/train.py scripts/
cp scripts-staging/evaluate.py scripts/
git add scripts/train.py scripts/evaluate.py
git commit -m "train.py and evaluate.py moved under .scripts folder"
```

Before wiring `dvc.yaml`, inspect the scripts to learn their real I/O:

- `train.py` reads `params.yaml`, reads `data/processed/clean.csv`, writes
  `models/model.pkl`, `data/processed/test_split.csv`, and **`metrics.json`
  in the project root** (`with open("metrics.json", "w")`).
- `evaluate.py` loads `models/model.pkl` and `data/processed/test_split.csv`
  and writes `reports/evaluation.json`. It does **not** read `metrics.json` —
  it recomputes the metrics itself, so `metrics.json` must not be a dep of
  `evaluate`.

### 3. Declare the `train` and `evaluate` stages

Final `dvc.yaml`:

```yaml
stages:
  ingest:
    cmd: python3 scripts/ingest.py
    deps:
      - scripts/ingest.py
      - data/raw/data.csv

  validate:
    cmd: python3 scripts/validate.py
    deps:
      - data/raw/data.csv
      - scripts/validate.py
    outs:
      - reports/validation.json:
          cache: false

  preprocess:
    cmd: python3 scripts/preprocess.py
    deps:
      - data/raw/data.csv
      - scripts/preprocess.py
    outs:
      - data/processed/clean.csv

  train:
    cmd: python3 scripts/train.py
    deps:
      - data/processed/clean.csv
      - scripts/train.py
    params:
      - n_estimators
      - max_depth
      - test_size
      - random_seed
    outs:
      - models/model.pkl
      - data/processed/test_split.csv
    metrics:
      - metrics.json:
          cache: false

  evaluate:
    cmd: python3 scripts/evaluate.py
    deps:
      - models/model.pkl
      - data/processed/test_split.csv
      - scripts/evaluate.py
    outs:
      - reports/evaluation.json:
          cache: false
```

Key points:

- `metrics.json` sits at the repository root because that is where
  `train.py` writes it — the declared path must match exactly.
- `metrics:` (not `outs:`) is used so `dvc metrics show` picks it up;
  `cache: false` keeps it Git-tracked.
- `reports/evaluation.json` is a regular `out` with `cache: false` —
  a small report meant to live in Git, not in the remote cache.

### 4. Reproduce the full pipeline

```bash
dvc repro
```

```text
Stage 'ingest' didn't change, skipping
Stage 'validate' didn't change, skipping
Stage 'preprocess' didn't change, skipping
Running stage 'train':
> python3 scripts/train.py
Trained: {'accuracy': 1.0, 'f1_score': 1.0}
Updating lock file 'dvc.lock'

Running stage 'evaluate':
> python3 scripts/evaluate.py
Evaluation: {'accuracy': 1.0, 'f1_score': 1.0, 'precision': 1.0, 'recall': 1.0, 'test_samples': 4}
Updating lock file 'dvc.lock'
```

Confirm the metric is registered:

```bash
dvc metrics show
```

```text
Path          accuracy    f1_score
metrics.json  1.0         1.0
```

### 5. Push the cache to SeaweedFS

```bash
dvc push
```

```text
Pushing
3 files pushed
```

The 3 files are the cached outputs: `clean.csv`, `test_split.csv`, and
`model.pkl`. (`metrics.json`, `validation.json`, and `evaluation.json` are
`cache: false`, so they are not pushed.)

### 6. Commit everything and tag the release

```bash
git add -A
git commit -m "complete DVC pipeline with train and evaluate stages"
git tag v1.0
```

The commit contains `dvc.yaml`, `dvc.lock`, `metrics.json`,
`reports/evaluation.json`, `reports/validation.json`, and the generated
`.gitignore` files — the DVC pointers, not the data blobs.

### 7. Verify

```bash
git status        # nothing to commit, working tree clean
git log --oneline # cc21e08 (HEAD -> main, tag: v1.0) ...
dvc status -c     # Cache and remote 's3' are in sync.
```

In the SeaweedFS Filer UI, open `/buckets/dvc-storage/files/md5/` — three
objects exist, one per pushed artefact:

| Path | Size | Artefact |
| --- | --- | --- |
| `files/md5/44/5bda81...` | 72 kB | `models/model.pkl` |
| `files/md5/d5/5df39c...` | 212 B | processed CSV |
| `files/md5/fb/235cd6...` | 702 B | processed CSV |

The Filer only shows content-hash names — that is expected. To map hashes
back to paths, use `dvc.lock` (each `out` records `md5`, `size`, `path`) or
`md5sum` on the local files:

```bash
grep -B2 -A3 "md5:" dvc.lock
md5sum models/model.pkl data/processed/clean.csv data/processed/test_split.csv
```

## Gotchas

### Output filename mismatches fail silently-ish

DVC does not check that the script wrote *something* — it checks that the
declared `outs` exist after the command finishes. `preprocess.py` printed
"Preprocessed: 20 clean rows" and exited 0, yet the stage failed because the
declared output was `cleaned.csv` while the script wrote `clean.csv`. Always
cross-check script I/O against `dvc.yaml`.

### `cache: false` outputs are Git files, not remote files

`metrics.json`, `reports/validation.json`, and `reports/evaluation.json` are
declared with `cache: false`: they stay in the workspace, appear in Git, and
are **not** pushed to SeaweedFS. Only the three cached artefacts show up in
the bucket.

### Scripts are Git deps, not DVC outs

`scripts/*.py` are listed under `deps:` so DVC re-runs a stage when code
changes. They live in `scripts-staging/` only as provisioning; they must be
copied to `scripts/` and committed — DVC does not version code.

### YAML indentation for out options

The `cache: false` option must be nested under the output path with correct
indentation:

```yaml
    outs:
      - reports/evaluation.json:
          cache: false
```

A misaligned `cache:` line produces `Mapping values are not allowed in this
context`.

## Command cheat sheet

```bash
# reproduce and inspect
dvc repro
dvc metrics show
dvc dag

# push to the SeaweedFS remote
dvc push
dvc status -c

# inspect recorded hashes
grep -B2 -A3 "md5:" dvc.lock
md5sum models/model.pkl data/processed/*.csv

# release
git add -A
git commit -m "complete DVC pipeline with train and evaluate stages"
git tag v1.0
git status
git log --oneline
```

## Further reading

- `dvc.yaml` stage fields — <https://doc.dvc.org/user-guide/project-structure/dvcyaml-files>
- `params` tracking — <https://doc.dvc.org/user-guide/project-structure/dvcyaml-files#parameters>
- Metrics and plots — <https://doc.dvc.org/user-guide/project-structure/dvcyaml-files#metrics-and-plots>
- `dvc push` / remotes — <https://doc.dvc.org/command-reference/push>
- SeaweedFS S3 API — the remote is configured as an S3-compatible endpoint at `http://localhost:8333`
