# Day 20 — Local MLflow Tracking Server on the Controlplane

## Problem

The xFusionCorp Industries ML team is in the process of adopting MLflow
for their experiment tracking. Your task is to set up a local MLflow
tracking server on the ML pipeline workstation, enabling the team to log
experiments from their training code.

MLflow is pre-installed on the controlplane. Launch the tracking server
in the background and choose the flags that satisfy every end-state
requirement below.

- The server listens on port 5000 and is reachable on all network
  interfaces, not only localhost.
- The backend store is a SQLite database at
  `/root/code/mlflow-backend/mlflow.db`. Create any parent directory
  first — MLflow aborts at startup if the backend directory is missing.
- The artifact root is `/root/code/mlflow-artifacts/`.
- The MLflow UI button at the top of the lab routes through the lab
  proxy, which reaches the server with a non-localhost host header and a
  different origin. Launch the server so it accepts any host header and
  any origin; otherwise the button returns a 403 or CORS error.
- The server process persists in the background so it survives terminal
  closure.
- Once the server is running, the Default experiment can be viewed from
  the MLflow UI button. The experiment is empty.

## Overview

Part of the *100 Days of MLOps* challenge. This day sets up a persistent
MLflow tracking server on the lab controlplane so the xFusionCorp ML team
can log experiments from training code. MLflow is pre-installed; the whole
task reduces to launching `mlflow server` in the background with the right
combination of flags — plus two non-obvious details (SQLite URI slashes and
shell job persistence) that decide whether the check passes.

## Task

| Fact | Value |
| --- | --- |
| Listen address | all interfaces (`0.0.0.0`), port `5000` |
| Backend store | SQLite at `/root/code/mlflow-backend/mlflow.db` |
| Artifact root | `/root/code/mlflow-artifacts/` |
| Access path | lab proxy → non-localhost Host header + different Origin |
| Lifecycle | background process that survives terminal closure |

Acceptance criteria:

- Server listens on port `5000` on all network interfaces, not only
  localhost.
- Backend store is the SQLite file above — its parent directory must exist
  before launch, otherwise MLflow aborts at startup.
- Artifact root is `/root/code/mlflow-artifacts/`.
- Server accepts **any Host header** and **any Origin**, so the lab's
  "MLflow UI" button (routed through a proxy) does not return 403/CORS
  errors.
- Process persists in the background and survives terminal closure.
- The `Default` experiment is visible from the MLflow UI button (and is
  empty).

## Solution

### 1. Create the directories

```bash
mkdir -p /root/code/mlflow-backend /root/code/mlflow-artifacts
```

Only the *directories* are needed — the `mlflow.db` file itself is created
by SQLite/MLflow on first use.

### 2. Launch the server

```bash
nohup mlflow server \
  --host 0.0.0.0 \
  --port 5000 \
  --allowed-hosts "*" \
  --cors-allowed-origins "*" \
  --backend-store-uri sqlite:////root/code/mlflow-backend/mlflow.db \
  --default-artifact-root /root/code/mlflow-artifacts/ \
  > /tmp/mlflow.log 2>&1 &

disown %1
```

Flag mapping:

| Requirement | Flag |
| --- | --- |
| All interfaces | `--host 0.0.0.0` |
| Port | `--port 5000` |
| SQLite backend store | `--backend-store-uri sqlite:////root/code/mlflow-backend/mlflow.db` |
| Artifact root | `--default-artifact-root /root/code/mlflow-artifacts/` |
| Any Host header / Origin (lab proxy) | `--allowed-hosts "*" --cors-allowed-origins "*"` |
| Survives terminal closure | `nohup ... &` (or `&` + `disown %1`) |

Note the **four slashes** in the SQLite URI — see Gotchas.

### 3. Trigger database creation and verify

In MLflow 3.x the SQLite file is created lazily — `/health` does not touch
the store, so force a real query:

```bash
sleep 3
curl -s http://localhost:5000/health        # → OK

curl -s -X POST http://localhost:5000/api/2.0/mlflow/experiments/search \
  -H 'Content-Type: application/json' -d '{"max_results": 10}'

ls -la /root/code/mlflow-backend/           # mlflow.db must be here
jobs                                        # empty → server disowned
ps aux | grep "[m]lflow server"             # process still running
```

Then open the **MLflow UI** button at the top of the lab: the interface
loads through the proxy and shows the empty `Default` experiment.

## Gotchas

### `sqlite:///path` is relative — use four slashes for absolute paths

The classic SQLAlchemy trap hit on this task:

- `sqlite:///root/code/mlflow-backend/mlflow.db` → **relative** path
  `root/code/mlflow-backend/mlflow.db` resolved against the server's
  working directory. Launched from `~/code`, the file ended up at
  `/root/code/root/code/mlflow-backend/mlflow.db` — silently "working",
  but failing the check.
- `sqlite:////root/code/mlflow-backend/mlflow.db` → **absolute** path
  `/root/code/mlflow-backend/mlflow.db`. This is the correct form.

Symptom: `ls /root/code/mlflow-backend/` empty while a stray `root/`
subtree appears under the launch directory. Fix: kill the server, remove
the stray tree, relaunch with the 4-slash URI.

### `&` alone does not survive terminal closure

A backgrounded job stays in the shell's job table and can receive `SIGHUP`
when the terminal closes. Either launch with `nohup cmd ... &` upfront, or
`disown %1` (equivalently `disown -h %1`) afterwards to detach the running
job — no restart needed. `jobs` must return empty afterwards.

### Redirects after a line-continuation become arguments

```bash
mlflow server ... --default-artifact-root /root/code/mlflow-artifacts/ \
> /tmp/mlflow.log 2>&1 &        # ← ERROR: passed to mlflow as an argument
```

A `\` at end of line glues the next line into the command, so `>` and the
log path were passed to Click: `Got unexpected extra argument
(/tmp/mlflow.log)`. Redirects and `&` belong on the last line *without* a
preceding backslash.

### `experiments/search` requires `max_results > 0`

`POST /api/2.0/mlflow/experiments/search` with `{}` returns
`INVALID_PARAMETER_VALUE: 'max_results' must be a positive integer` — send
`{"max_results": 10}` (the UI does this automatically).

### Wildcard CORS warnings are harmless

`MLFLOW_SERVER_CORS_ALLOWED_ORIGINS=* ... disabling credentialed CORS` is
expected: `*` only disables *credentialed* cross-origin requests, which
the lab proxy does not need.

## Command cheat sheet

```bash
# prerequisites
mkdir -p /root/code/mlflow-backend /root/code/mlflow-artifacts

# launch (4-slash absolute SQLite URI) + detach
nohup mlflow server --host 0.0.0.0 --port 5000 \
  --allowed-hosts "*" --cors-allowed-origins "*" \
  --backend-store-uri sqlite:////root/code/mlflow-backend/mlflow.db \
  --default-artifact-root /root/code/mlflow-artifacts/ \
  > /tmp/mlflow.log 2>&1 &
disown %1

# verify
curl -s http://localhost:5000/health
curl -s -X POST http://localhost:5000/api/2.0/mlflow/experiments/search \
  -H 'Content-Type: application/json' -d '{"max_results": 10}'
ls -la /root/code/mlflow-backend/
jobs; ps aux | grep "[m]lflow server"
tail -f /tmp/mlflow.log
```

## Further reading

- `mlflow server` CLI options — <https://mlflow.org/docs/latest/cli.html#mlflow-server>
- Tracking server & backend stores — <https://mlflow.org/docs/latest/ml/tracking/server>
- SQLAlchemy SQLite URL forms (`sqlite:///` vs `sqlite:////`) — <https://docs.sqlalchemy.org/en/20/core/engines.html#sqlite>
- Bash job control (`disown`, `nohup`) — `man bash`, "JOB CONTROL" section
