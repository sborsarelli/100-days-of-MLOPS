# MLflow 3.x — Quick Start per le prove CodeKloud 100-Days-of-MLOps

## I due libri del 2021 sono ancora utili?

**Concetti: sì. Sintassi/comandi: no, non alla lettera.**

I libri coprono MLflow 1.x (metà 2021). Da allora ci sono state tre major release
(2.0 → 2.x → 3.x) con breaking change significative. Quello che resta valido:

- Il **Tracking** (experiment, run, param, metric, tag, artifact)
- Il **Model Registry** (register, stage: None→Staging→Production→Archived,
  transition_model_version_stage)
- Il **Model Serving** e i **flavor** (sklearn, pytorch, pyfunc, onnx, etc.)
- **MLflow Projects**, **autologging**, **Evaluate**, firma/risposte dei modelli
- La distinzione artifact store / metadata store / registry

Quello che è **cambiato** e ti farà fallire un task se copiato dai libri:

| MLflow 1.x (libri)                | MLflow 3.x (oggi)                                        |
|-----------------------------------|----------------------------------------------------------|
| `mlflow ui`                       | `mlflow server` (il comando `ui` è rimosso/deprecato)    |
| `mlflow server --serve-artifacts` | `mlflow server --serve-artifacts` ancora valido, ma controlla i flag |
| `log_model(artifact_path="model")`| `log_model(name="model")` — `artifact_path` deprecato    |
| Modelli come artifact del run     | **LoggedModels**: entità di prima classe, pagina dedicata in UI |
| `run_info.run_uuid`               | `run_info.run_id`                                        |
| Registry richiede backend SQL     | File store supporta registry; usa comunque `sqlite:///mlflow.db` nei task |
| Flavor `fastai`, `mleap`, `h2o`   | Rimossi — usa `pyfunc`/`sklearn`/`onnx`                  |
| `mlflow recipes` / `pipelines`    | Rimossi — usa Projects + tracking manuale                |
| Tag git `mlflow.gitBranchName`    | Rimossi                                                     |

**Regola pratica**: leggi i libri per capire *cosa* tracciare e *perché*; per
*scrivere il codice* copia solo dalla documentazione attuale.

---

## Comandi essenziali (MLflow 3.x)

```bash
pip install "mlflow>=3.1" "mlflow-skinny[extras]"   # skinny per meno dipendenze

# Avvia il tracking server (NON `mlflow ui`)
mlflow server \
    --backend-store-uri sqlite:///mlflow.db \
    --default-artifact-root ./mlruns \
    --host 0.0.0.0 --port 5000
# UI su http://localhost:5000

# Oppure file store locale (solo per esperimenti veloci, niente registry affidabile)
# mlflow.set_tracking_uri("file:///path/to/mlruns")
```

## Tracking (fluent API — invariata dai libri)

```python
import mlflow
from sklearn.ensemble import RandomForestClassifier

mlflow.set_tracking_uri("http://127.0.0.1:5000")   # o "sqlite:///mlflow.db"
mlflow.set_experiment("fraud-detection")

with mlflow.start_run(run_name="rf-v1") as run:
    mlflow.log_param("n_estimators", 100)
    mlflow.log_param("max_depth", 6)

    model = RandomForestClassifier(n_estimators=100, max_depth=6)
    model.fit(X_train, y_train)

    acc = model.score(X_test, y_test)
    mlflow.log_metric("accuracy", acc)
    mlflow.log_artifact("metrics.json")

    # MLflow 3: `name` invece di `artifact_path`
    mlflow.sklearn.log_model(model, name="fraud-model",
                             registered_model_name="fraud-detector")
```

## Model Registry

```python
from mlflow import MlflowClient

client = MlflowClient()

# Cerca modelli
client.search_registered_models()
client.search_model_versions("name='fraud-detector'")

# Stage: None -> Staging -> Production -> Archived
client.transition_model_version_stage(
    name="fraud-detector",
    version=1,
    stage="Production",
    archive_existing_versions=True,
)

# Carica la versione in produzione
model = mlflow.pyfunc.load_model(
    "models:/fraud-detector/Production"
)
```

## Autologging (il corso lo usa spesso)

```python
import mlflow.sklearn
mlflow.sklearn.autolog()
model.fit(X_train, y_train)   # logga metriche/parametri automaticamente
```

## Serve il modello

```bash
mlflow models serve \
    -m "models:/fraud-detector/Production" \
    --host 0.0.0.0 --port 5001
# REST: POST /invocations  con JSON {"inputs": ...} o {"dataframe_split": ...}
```

---

## Documentazione ufficiale da studiare (in ordine)

Sito: https://mlflow.org/docs/latest/ (sezione "MLflow AI Platform")

1. **Quickstart** — https://mlflow.org/docs/latest/ml/mlflow-quickstart/
   Per vedere l'intero flusso tracking → registry in 10 minuti.

2. **MLflow Tracking** — https://mlflow.org/docs/latest/ml/tracking/
   - Concepts (experiment, run, params, metrics, tags, artifacts)
   - Fluent API (`mlflow.start_run`, `log_param`, `log_metric`, `log_artifact`, `log_model`)
   - **MLflow Client** per query/registry
   - **Search Runs / search_model_versions** (serve nei task di confronto)

3. **MLflow Models** — https://mlflow.org/docs/latest/ml/model/ (sezione "Models")
   - `log_model` con `name=` (attenzione: non più `artifact_path`)
   - **LoggedModels** (novità MLflow 3)
   - pyfunc, flavor sklearn, `models:/` URI, `mlflow models serve`

4. **Model Registry** — https://mlflow.org/docs/latest/ml/model-registry/
   - register_model, versioning, stage transitions, alias/webhooks
   - Nel corso CodeKloud il task classico è: loggare → registrare →
     portare in Staging/Production → caricare con `models:/nome/stage`

5. **MLflow Projects** — https://mlflow.org/docs/latest/ml/projects/
   - `MLproject` file, `mlflow run`, params passati da CLI
   - (`mlflow recipes` non esiste più: se un video vecchio lo usa, salta)

6. **MLflow Evaluate** — https://mlflow.org/docs/latest/genai/eval-monitor/ oppure
   `/docs/latest/ml/model-evaluation/` per evaluate classico
   - `mlflow.evaluate(model, data, targets)` per metriche di qualità

7. **MLflow 3 Migration Guide** — https://mlflow.org/docs/latest/ml/mlflow-3/
   - Lettura obbligata: è la lista ufficiale dei breaking change

8. **CLI di MLflow** — `mlflow --help`, `mlflow server --help`
   - Verifica sempre i flag esatti: `--backend-store-uri`,
     `--default-artifact-root`, `--serve-artifacts`, `--artifacts-destination`

---

## Trappole tipiche nei lab CodeKloud con MLflow 3

- **`mlflow ui` non esiste più** → usa `mlflow server` (serve `--backend-store-uri`
  per il registry persistente; altrimenti perde tutto al riavvio).
- **`artifact_path` deprecato** → usa `name=` in `log_model`.
- **Registry con file store**: per stage transitions affidabili usa
  `sqlite:///mlflow.db` come backend store.
- **Artifact root**: con `mlflow server` gli artifact vanno in
  `--default-artifact-root`; con file store locale sotto `mlruns/`.
- **Versioni pin**: se un `requirements.txt` del lab fissa `mlflow==1.x`,
  aggiorna a `mlflow>=3.0` (o la versione richiesta dal task) —
  i vecchi pin falliscono su Python moderno.
- **Docker**: se il task usa un'immagine con MLflow vecchio, controlla la
  versione nell'immagine prima di scrivere codice.
- **UI**: in MLflow 3 gli artifact dei modelli non sono più nella tab
  "Artifacts" del run — sono nella pagina **Logged Models**.
