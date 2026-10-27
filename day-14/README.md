# Day 14 — Defining a Two-Stage DVC Pipeline

## Overview

Part of the *100 Days of MLOps* challenge. Days 12–13 fixed the storage side of DVC
(`dvc push` / `dvc pull` against a SeaweedFS remote); this day moves to the compute side:
turning two loose Python scripts into a reproducible **pipeline** declared in `dvc.yaml`.

The xFusionCorp Industries ML team has the `fraud-detection` project at
`/root/code/fraud-detection/` with DVC already initialised. The processing script
(`src/data/process_data.py`), the splitting script (`src/data/split_data.py`) and the raw
input (`data/raw/transactions.csv`) are all in place — but nothing tells DVC how they
relate. The goal is to declare a two-stage pipeline so the whole flow reproduces with a
single `dvc repro`, and so `dvc dag` can show how the stages chain together.

The write-up documents the pipeline definition, why the same file can appear both as a
`-d` dependency and as a command argument (and when each is needed), the verification of
all acceptance criteria, and pointers for deeper study.

## Task

The environment:

| Fact | Value |
| --- | --- |
| Project | `/root/code/fraud-detection/` (DVC initialised) |
| Scripts | `src/data/process_data.py`, `src/data/split_data.py` |
| Raw input | `data/raw/transactions.csv` |
| Constraint | Do not modify the Python files or the input data |

Acceptance criteria:

- A `dvc.yaml` defines two stages:
  - **`process_data`** — runs `python3 src/data/process_data.py`; depends on
    `data/raw/transactions.csv` and `src/data/process_data.py`; produces
    `data/processed/clean_transactions.csv`.
  - **`split_data`** — runs `python3 src/data/split_data.py`; depends on
    `data/processed/clean_transactions.csv` (the upstream stage's output, so DVC chains
    the stages) and `src/data/split_data.py`; produces `data/processed/train.csv` and
    `data/processed/test.csv`.
- The pipeline has been reproduced so both stages execute in order and `dvc.lock` is
  written, and `dvc status` reports no stale stages.
- Use `python3` (not `python`) in the stage commands. Once the pipeline is valid,
  `dvc dag` prints the dependency graph showing how the two stages chain together.

## Background: how a DVC stage works

### A stage is a Makefile rule, versioned

`dvc stage add` records in `dvc.yaml` three things about a processing step:

- **`cmd`** — the shell command to run;
- **`deps`** — the files the stage *reads* (code, data, config);
- **`outs`** — the files the stage *writes*.

DVC hashes every dependency and output. When `dvc repro` runs, it re-executes only the
stages whose inputs changed — exactly like `make`, but content-hashed instead of
timestamp-based, and with results recorded in `dvc.lock`.

### `-d` is for DVC, arguments are for the script

The same file can appear twice in a `dvc stage add` invocation for two different reasons:

```bash
dvc stage add -n prepare \
  -d data/data.xml \              # declared to DVC (hashed, tracked, drawn in the DAG)
  -o data/prepared \
  python src/prepare.py data/data.xml
#                       ^-- passed to Python as sys.argv[1] at runtime
```

- `-d` tells **DVC** the file is a dependency. DVC never parses the command line to
  guess dependencies — you declare them explicitly.
- The trailing `data/data.xml` is what **the script** receives. It is required only
  because the official tutorial's `prepare.py` reads its input from `sys.argv[1]`
  (and exits with an error if it is missing).

The two channels are independent: a script that hardcodes its paths needs no argument
— and that is exactly our case here, since `process_data.py` opens
`data/raw/transactions.csv` directly:

```python
df = pd.read_csv("data/raw/transactions.csv")
df.to_csv("data/processed/clean_transactions.csv", index=False)
```

The acceptance criteria confirm it: the stage's `cmd` must be literally
`python3 src/data/process_data.py`, with no extra arguments.

### Chaining: an upstream output becomes a downstream dependency

Stages connect into a DAG when one stage's `deps` reference another stage's `outs`.
Putting `data/processed/clean_transactions.csv` in `split_data`'s `deps` is what makes
`dvc dag` draw `process_data` → `split_data` — and what makes `dvc repro` order them
correctly. Declaring the script files themselves as `deps` is good practice too: editing
the code then correctly marks the stage stale.

### `dvc.lock` records what actually ran

`dvc repro` writes `dvc.lock` with the hash of every dep and out at run time. Committing
`dvc.yaml` + `dvc.lock` to Git is what lets anyone reproduce the exact same pipeline
later — while the data payloads stay out of Git, tracked by DVC instead.

## Solution

Run from `/root/code/fraud-detection` (the DVC project root — the scripts use relative
paths, so the working directory matters).

### 1. Declare the `process_data` stage

```bash
dvc stage add -n process_data \
  -d src/data/process_data.py -d data/raw/transactions.csv \
  -o data/processed/clean_transactions.csv \
  python3 src/data/process_data.py
```

### 2. Declare the `split_data` stage

```bash
dvc stage add -n split_data \
  -d src/data/split_data.py -d data/processed/clean_transactions.csv \
  -o data/processed/train.csv -o data/processed/test.csv \
  python3 src/data/split_data.py
```

The key point: `clean_transactions.csv` goes in `-d` here, not `-o` — it is produced by
the upstream stage, and that cross-reference is what builds the pipeline.

### 3. Reproduce the pipeline

```bash
dvc repro
```

This executes both stages in dependency order and writes `dvc.lock`.

## Resulting `dvc.yaml`

```yaml
stages:
  process_data:
    cmd: python3 src/data/process_data.py
    deps:
    - data/raw/transactions.csv
    - src/data/process_data.py
    outs:
    - data/processed/clean_transactions.csv
  split_data:
    cmd: python3 src/data/split_data.py
    deps:
    - data/processed/clean_transactions.csv
    - src/data/split_data.py
    outs:
    - data/processed/test.csv
    - data/processed/train.csv
```

(The order of `deps`/`outs` entries is irrelevant — what matters is
`clean_transactions.csv` being an `out` of the first stage and a `dep` of the second.)

## Verification

**1. Both stages run in order**

```console
$ dvc repro
Running stage 'process_data':
> python3 src/data/process_data.py
Processed 15 rows
Updating lock file 'dvc.lock'
Running stage 'split_data':
> python3 src/data/split_data.py
Train: 12 rows, Test: 3 rows
Updating lock file 'dvc.lock'
```

**2. No stale stages**

```console
$ dvc status
Data and pipelines are up to date.
```

**3. The DAG shows the chain**

```console
$ dvc dag
+--------------+
| process_data |
+--------------+
        *
        *
        *
 +------------+
 | split_data |
 +------------+
```

## Gotchas

### `Could not create .gitignore entry` during `stage add`

If the output directory (`data/processed/`) does not exist yet, `dvc stage add` warns
that it could not add the outputs to `.gitignore`. Harmless: DVC retries when the stage
first runs, which is why `git add data/processed/.gitignore` appears after `dvc repro`.

### `Unable to find 'less'` in `dvc dag`

`dvc dag` pages its output through `less` when available; on a minimal container it
falls back to plain ASCII output. The graph is still printed correctly — the warning can
be ignored (or install `less` to silence it).

### Passing the input path as an argument anyway

Adding `data/raw/transactions.csv` after `python3 src/data/process_data.py` would run
fine (Python ignores unneeded `sys.argv` entries here), but it changes the recorded
`cmd` away from what the task specifies — and if the script ever gains `argparse`
validation it would break. Declare files with `-d`; pass arguments only when the script
reads them.

### Commit the pipeline files

`dvc.yaml`, `dvc.lock` and the generated `.gitignore` are the pieces Git should track —
they are what makes `dvc repro` reproducible on another clone (combined with `dvc pull`
for the data, as in Day 13):

```bash
git add dvc.yaml dvc.lock data/processed/.gitignore
```

## Further reading

### Tutorial

- **Get Started: Data Pipelines** — the official DVC tutorial,
  <https://doc.dvc.org/start/data-pipelines/data-pipelines>. It walks through
  `dvc stage add` option by option (`-n`, `-d`, `-o`, `-p` for params), shows the
  generated `dvc.yaml`, and chains `prepare → featurize → train → evaluate` into a real
  DAG — the same pattern used here with two stages. Note how its `prepare.py` takes the
  input path from `sys.argv`, which is why `data/data.xml` appears both as `-d` and as a
  command argument in its example.

### Books

- **Open Source MLOPs with DVC and CML** — Matthew Upson (Packt, 2025). The book-length
  treatment of exactly this topic: Chapter 6, *"Building pipelines with DVC"*, covers
  `dvc stage add`, `dvc.yaml`, `deps`/`outs` and `dvc repro`; Chapter 7,
  *"Advanced Pipelines — Parameterization and foreach Stages"*, goes into `params.yaml`
  and advanced stage definitions.
- **The Pragmatic Programmer for Machine Learning** — Marco Scutari & Mauro Malvestio
  (Chapman & Hall/CRC, 2023). **Open access** at <https://ppml.dev/>. Chapter 11,
  *"Tools to Manage Pipelines in Production"*, discusses DVC among the lightweight,
  Git-based pipeline tools, and the authors' companion material uses `dvc stage`,
  `dvc repro` and `dvc dag` in its running example.
- *Note:* a frequently suggested alternative, *Practical MLOps* (Gift & Deza,
  O'Reilly 2021), does **not** actually cover DVC — its dataset-versioning discussion
  is about Azure ML datasets (Chapter 8), and DVC appears neither in the table of
  contents nor in the companion repository.

### Official documentation

- `dvc stage add` — <https://doc.dvc.org/command-reference/stage/add> — every flag used
  above (`-n`, `-d`, `-o`, `-p`).
- `dvc repro` — <https://doc.dvc.org/command-reference/repro> — stage ordering, caching
  and `dvc.lock`.
- `dvc dag` — <https://doc.dvc.org/command-reference/dag> — the dependency graph view.
- `dvc status` — <https://doc.dvc.org/command-reference/status> — detecting stale stages.
- `dvc.yaml` file format — <https://doc.dvc.org/user-guide/project-structure/dvcyaml-files>
  — the schema behind `stages:`.
