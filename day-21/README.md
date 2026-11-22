# Day 21 — Logging a Training Run to the MLflow Tracking Server

## Problem

A xFusionCorp Industries data scientist needs a training run recorded in
MLflow so the team has a baseline record on the tracking dashboard. The
non-MLflow scaffolding has already been written at
`/root/code/log_experiment.py`; the MLflow logging calls are left as TODO
blocks. Your task is to complete the script so that every element of the
run is captured by the MLflow tracking server.

The MLflow tracking server is already running on port `5000`. The MLflow
UI button at the top of the lab can be opened to view the dashboard; the
Default experiment is present on first load.

`/root/code/log_experiment.py` can be opened in the VS Code editor. The
script prepares a params dictionary, fits a trivial sklearn model, and
advertises a pair of synthetic evaluation scores (`accuracy` and `f1`).
Three blocks marked `# TODO` inside the `mlflow.start_run()` context are
the only edits required.

Execute the script once (`python3 /root/code/log_experiment.py`) after
the TODOs are completed. The end state must include:

- Exactly one new run in the Default experiment.
- Every hyperparameter in the params dict (`n_estimators=100`,
  `max_depth=5`, `random_state=42`) recorded as a run parameter.
- Both advertised scores (`accuracy`, `f1_score`) recorded as run
  metrics.
- The sklearn model captured as an MLflow model artefact on the run.

The result can be confirmed in the MLflow UI—once the run is opened, the
Parameters, Metrics, and Artifacts panels each show the expected content.

## Overview

Part of the *100 Days of MLOps* challenge. With the tracking server from
Day 20 already running on port 5000, this day completes a half-written
script — `/root/code/log_experiment.py` — that trains a trivial
`DummyClassifier` on a synthetic fixture and must record the run in
MLflow. The only edits are three `# TODO` blocks inside the
`mlflow.start_run()` context: parameters, metrics, and the model
artefact.

## Task

| Fact | Value |
| --- | --- |
| Script | `/root/code/log_experiment.py` |
| Tracking URI | `http://localhost:5000` (server from Day 20) |
| Model | `DummyClassifier(strategy="most_frequent")` fit on 4 rows |
| Params dict | `n_estimators=100, max_depth=5, random_state=42` |
| Metrics | `accuracy` (0.75), `f1_score` (~0.857), computed from predictions |

Acceptance criteria:

- Exactly **one** new run in the `Default` experiment.
- Every entry in `params` recorded as a run parameter.
- Both scores recorded as metrics named `accuracy` and `f1_score`.
- The sklearn model captured as an MLflow model artefact on the run.
- Parameters, Metrics, and Artifacts panels populated in the UI.

## Solution

The scaffolding already sets the tracking URI, builds the params dict,
fits the model, and computes the scores from its own predictions. The
three TODOs map one-to-one onto the MLflow logging API:

```python
with mlflow.start_run():

    # TODO 1: log every entry in `params` as an MLflow parameter
    mlflow.log_params(params)

    # TODO 2: log `accuracy` and `f1` as metrics named
    # "accuracy" and "f1_score"
    mlflow.log_metric("accuracy", accuracy)
    mlflow.log_metric("f1_score", f1)

    # TODO 3: log the trained `model` as an MLflow sklearn model artefact
    mlflow.sklearn.log_model(sk_model=model, name="Dummy_Model")

    print(f"accuracy={accuracy}, f1_score={f1}")
```

API choices:

- `mlflow.log_params(dict)` logs the whole dict in one call —
  equivalent to looping `mlflow.log_param(k, v)` but idiomatic.
- Metric **names** must match the task exactly (`"f1_score"`, not `"f1"`).
- `mlflow.sklearn.log_model` serialises the estimator (pickle +
  `MLmodel` descriptor + inferred `requirements.txt`) into the run's
  artefact store.

### Run once

```bash
python3 /root/code/log_experiment.py
```

```text
accuracy=0.75, f1_score=0.8571428571428571
🏃 View run clumsy-penguin-756 at: http://localhost:5000/#/experiments/0/runs/5a64a485437b45f993bfdedeef04255e
🧪 View experiment at: http://localhost:5000/#/experiments/0
```

### Verify

Open the run in the MLflow UI:

- **Parameters** → `n_estimators`, `max_depth`, `random_state`
- **Metrics** → `accuracy = 0.75`, `f1_score ≈ 0.857`
- **Artifacts** → `Dummy_Model/` containing `MLmodel`, `model.pkl`,
  `requirements.txt`, `conda.yaml`

## Gotchas

### `log_model` second argument: `name` (3.x) vs `artifact_path` (2.x)

In MLflow 3.x the flavour `log_model` functions take `name=` for the
artefact subdirectory (the old `artifact_path=` is deprecated). On a 2.x
client the same call raises `unexpected keyword argument 'name'` — use
`artifact_path="Dummy_Model"` there. Check with `mlflow --version`.

### "Exactly one new run" means run the script once

Every execution of the script creates a run — including exploratory or
debug runs. If the script was executed multiple times, delete the extra
runs from the UI before the check validates the end state.

### The torch `+cpu` warning is harmless

`Found torch version (2.14.1+cpu) contains a local version label` —
MLflow strips the `+cpu` local label when pinning `requirements.txt`
for the logged model, because local labels are not installable from
PyPI. The warning does not affect the run, the artefact, or the check.

### Metrics must come from the model, not constants

The scaffolding computes `accuracy`/`f1` from `model.predict(X_fit)` on
purpose: the lab grades the *logging API*, and deriving scores keeps the
run honest and deterministic (0.75 / 0.857 on this fixture).

## Command cheat sheet

```bash
# run once, then stop — each execution creates a run
python3 /root/code/log_experiment.py

# inspect from the CLI instead of the UI
mlflow runs list --experiment-id 0 --tracking-uri http://localhost:5000
curl -s -X POST http://localhost:5000/api/2.0/mlflow/runs/search \
  -H 'Content-Type: application/json' \
  -d '{"experiment_ids": ["0"], "max_results": 10}'

# artefact tree on disk (server-side, from Day 20)
find /root/code/mlflow-artifacts -name model.pkl
```

## Further reading

- `mlflow.log_params` / `log_metric` — <https://mlflow.org/docs/latest/api_reference/python_api/mlflow.html>
- `mlflow.sklearn.log_model` — <https://mlflow.org/docs/latest/api_reference/python_api/mlflow.sklearn.html>
- Tracking runs, params, metrics, artefacts — <https://mlflow.org/docs/latest/ml/tracking/>
