# MLflow 1.x/2.x → 3.x: come riscrivere comandi e codice dei libri 2021

Questa guida serve a **tradurre** ciò che trovi nei due PDF del 2021
(`Beginning MLOps with MLflow`, Apress; `Machine Learning Engineering with MLflow`,
Packt — entrambi scritti per MLflow 1.x) nella sintassi corretta per **MLflow 3.x**.

Principio: la *semantica* è invariata (experiment/run/param/metric/artifact/registry);
cambiano nomi di parametri, comandi CLI e concetti di UI. Sotto trovi la tabella di
traduzione + i capitoli dei libri utili per le prove.

---

## 1. CLI: comandi modificati o rimossi

| Vecchio (libri 2021) | Attuale (MLflow 3.x) | Note |
|---|---|---|
| `mlflow ui` | `mlflow server` | `ui` è deprecato/rimosso. `mlflow server` avvia UI + REST API + job scheduler |
| `mlflow ui --backend-store-uri sqlite:///mlflow.db --default-artifact-root ./mlruns` | `mlflow server --backend-store-uri sqlite:///mlflow.db --default-artifact-root ./mlruns --host 0.0.0.0 --port 5000` | Stessi flag principali; aggiungi sempre host/port espliciti |
| `mlflow server --serve-artifacts --artifacts-destination ./artifacts` | invariato | Serve gli artifact via UI senza storage esterno |
| `mlflow run <uri> -P lr=0.01` | `mlflow run <uri> -P lr=0.01` | Invariato, ma `mlflow recipes` non esiste più |
| `mlflow models serve -m runs:/<run_id>/model` | `mlflow models serve -m models:/<name>/<version>` oppure `runs:/<run_id>/<artifact_path>` | Meglio usare `models:/` con LoggedModels |
| `mlflow deployments ...` (deployment server) | RIMOSSO | Usa `mlflow models serve` o Docker |
| `mlflow start-server` | RIMOSSO | Vedi sopra |
| `mlflow experiments ...` / `mlflow runs ...` | `mlflow experiments search`, `mlflow runs search` | Verifica `mlflow <cmd> --help`: la CLI è stata riorganizzata |

**Regola d'oro**: prima di usare un comando dei libri, lancia
`mlflow <comando> --help` — la CLI 3.x è la fonte di verità.

---

## 2. Tracking API (fluent) — quasi invariata

```python
# === INVARIATO: usa così com'è ===
import mlflow

mlflow.set_tracking_uri("sqlite:///mlflow.db")   # o http://host:5000
mlflow.set_experiment("experiment-name")

with mlflow.start_run(run_name="my-run") as run:
    mlflow.log_param("lr", 0.01)          # ok
    mlflow.log_params({"a": 1, "b": 2})   # ok
    mlflow.log_metric("acc", 0.95)        # ok
    mlflow.log_metrics({"acc": 0.95})     # ok
    mlflow.log_artifact("file.csv")       # ok
    mlflow.set_tag("author", "me")        # ok

# === CAMBIATO ===
run.info.run_uuid          # 1.x/2.x  ->  run.info.run_id        # 3.x
```

**Attenzione ai tag git**: `mlflow.gitBranchName`, `mlflow.gitRepoURL`,
`mlflow.gitCommit` **non vengono più registrati** automaticamente in MLflow 3.
Se un task verifica questi tag, settali a mano con `mlflow.set_tag()`.

---

## 3. log_model / log_artifact — la modifica più importante

I libri usano `artifact_path` come nome del parametro. In MLflow 3 è deprecato
e sostituito da `name`. Il codice dei libri:

```python
# === VECCHIO (libri) — NON copiare ===
mlflow.sklearn.log_model(
    sk_model=model,
    artifact_path="model",                    # <- deprecato
    conda_env="conda.yaml",
    registered_model_name="fraud-detector",
)
```

```python
# === ATTUALE (MLflow 3.x) ===
mlflow.sklearn.log_model(
    model=model,                              # arg rinominato in alcuni flavor
    name="model",                             # era artifact_path
    registered_model_name="fraud-detector",   # ok
    input_example=X_sample,                   # raccomandato: genera la signature
)
# oppure senza run context (MLflow 3 supporta LoggedModels senza start_run):
# mlflow.sklearn.log_model(model, name="fraud-detector", registered_model_name=...)
```

Tabella di traduzione per i flavor più usati nei libri:

| Vecchio argomento | Nuovo (3.x) |
|---|---|
| `artifact_path="model"` | `name="model"` |
| `sk_model=` (sklearn) | `model=` (o controlla la signature esatta del flavor) |
| `python_model=` (pyfunc) | `python_model=` invariato |
| `conda_env=` / `python_env=` | Preferisci `model_config=` / `pip_requirements=`; `python_env` legacy ancora accettato in molti flavor |
| `mlflow.log_artifacts("dir", artifact_path="x")` | `mlflow.log_artifacts("dir", artifact_path="x")` — qui `artifact_path` resta valido per i file normali |

> **Sottigliezza**: `artifact_path` è deprecato solo per `log_model`. Per
> `log_artifact`/`log_artifacts`/`log_dict` il nome è ancora `artifact_path`.

---

## 4. Model Registry — API stabile, URI cambiati

```python
# === INVARIATO ===
from mlflow import MlflowClient
client = MlflowClient()

client.create_registered_model("fraud-detector")
client.transition_model_version_stage("fraud-detector", 1, "Staging")
client.transition_model_version_stage("fraud-detector", 1, "Production",
                                      archive_existing_versions=True)
client.search_model_versions("name='fraud-detector' AND tags.env='prod'")

# === CARICAMENTO: i URI `models:/` restano, maattenzione alle alias (novità) ===
mlflow.sklearn.load_model("models:/fraud-detector/Production")   # stage — ok
mlflow.sklearn.load_model("models:/fraud-detector/1")            # versione — ok
mlflow.sklearn.load_model("models:/fraud-detector@champion")     # alias — nuovo in 3.x
mlflow.sklearn.load_model("runs:/<run_id>/model")                # run artifact — ok
```

**Novità MLflow 3 — LoggedModels**: un modello loggato con `name=` diventa un
LoggedModel ricercabile (`mlflow.search_logged_models()` /
`client.search_logged_models()`), non solo un artifact del run. Nella UI i modelli
sono in una pagina dedicata, **non più** nella tab Artifacts del run.

**Registry su file store**: con `mlruns` locale il registry funziona ma è meno
robusto; per i task con stage transitions usa `sqlite:///mlflow.db` come
backend store (è quello che il corso CodeKloud si aspetta).

---

## 5. Autologging — quasi invariato

```python
# I libri: mlflow.sklearn.autolog() / mlflow.tensorflow.autolog() / mlflow.pytorch.autolog()
# MLflow 3: stessi entry point. Cambia solo:
#   - TensorFlow: parametro `every_n_iter` rimosso dall'autolog
#   - Alcuni flavor minori rimossi (fastai, mleap, h2o, gluon, diviner)
```

Se un libro usa `mlflow.fastai.autolog()` o `mlflow.mleap.log_model()` →
sostituisci con `mlflow.pyfunc` + wrapper manuale.

---

## 6. Projects — invariato nella sostanza

```yaml
# MLproject — file identico ai libri
name: fraud-detection
conda_env: conda.yaml        # accettato ancora; puoi usare python_env o dependencies
entry_points:
  main:
    parameters:
      lr: {type: float, default: 0.01}
    command: "python train.py --lr {lr}"
```

```bash
mlflow run . -P lr=0.01        # ok
mlflow run <git-uri>           # ok
```

Unica differenza: `mlflow recipes`/`pipelines` (che alcuni esempi online chiamano
"pipelines") **non esistono più** — usa solo Projects.

---

## 7. Serving — comandi invariati, path diversi

```bash
# Vecchio stile (ancora valido)
mlflow models serve -m runs:/<run_id>/model -p 5001 --no-conda

# Stile moderno consigliato per i task
mlflow models serve -m "models:/fraud-detector/Production" -p 5001 --no-conda

# Query
curl http://127.0.0.1:5001/invocations -H 'Content-Type: application/json' \
     -d '{"dataframe_split": {"columns": ["f1","f2"], "data": [[1.0,2.0]]}}'
```

In MLflow 3 il formato di default delle previsioni dipende dalla signature del
modello; i libri usano dataframe — continua a usare `dataframe_split` o
`inputs` come da signature.

---

## 8. UI: cosa è cambiato visivamente

| MLflow 1.x/2.x UI | MLflow 3.x UI |
|---|---|
| Tab "Artifacts" nel run con `model/` dentro | Modelli in **Logged Models** separati; tab Artifacts mostra solo file/artifact non-model |
| Model Registry nel menu a sinistra | Registry + LoggedModels; stage come prima |
| Esperimenti come cartelle in `mlruns` | Stessa logica, più filtri/colonne |

Nei task che dicono "apri la UI e verifica X", la verifica è quasi sempre sul
**Model Registry** (versioni, stage) o sugli **experiment** — non sugli artifact
del run.

---

## 9. Elenco dei capitoli dei due libri da studiare

### Libro A — *Beginning MLOps with MLflow* (Alla & Adari, Apress 2021)

| Capitolo | Titolo | Da studiare? | Perché |
|---|---|---|---|
| 1 | Getting Started: Data Analysis | Opzionale | Dataset e analisi, non MLflow |
| 2 | Building Models | Opzionale | sklearn/PySpark + cross-validation, base per i task |
| 3 | What Is MLOps? | **Sì — concetti** | Definizioni MLOps, cicli di vita: utili per task "spiega/completa" |
| 4 | Introduction to MLFlow | **Sì — fondamentale** | Experiment, logging di metriche/parametri/artifact, log_model, deploy locale e query. È il cuore MLflow del libro: rileggi con la tabella di traduzione sopra |
| 5 | Deploying in AWS | Parziale | Solo se il task tocca SageMaker; i concetti (artifact in S3, docker image) si riusano con altri storage |
| 6 | Deploying in Azure | Parziale | Come sopra |
| 7 | Deploying in Google | Parziale | Come sopra |

> NB: il capitolo 4 è scritto per MLflow 1.x (~metà 2021): `mlflow ui`,
> `artifact_path`, `conda_env` ovunque. **Non copiare i comandi**: usa le sezioni
> 1–7 di questa guida come sostituzione.

### Libro B — *Machine Learning Engineering with MLflow* (Lauchande, Packt 2021)

| Capitolo | Titolo | Da studiare? | Perché |
|---|---|---|---|
| 1 | Introducing MLflow | **Sì — fondamentale** | Architettura MLflow, moduli (tracking/models/registry/projects), primi esempi |
| 2 | Your Machine Learning Project | Sì (metà) | Framing del problema: come strutturare un progetto ML end-to-end |
| 3 | Your Data Science Workbench | Parziale | Setup ambiente riproducibile — utile se i task chiedono env/conda |
| 4 | Experiment Management in MLflow | **Sì — fondamentale** | Experiments, hyperparameter tuning tracciato: esattamente il pattern dei task CodeKloud |
| 5 | Managing Models with MLflow | **Sì — fondamentale** | Model flavors, **signature/schema** del modello, model registry, lifecycle. Le signature sono il punto più "sensitive" alle versioni |
| 6 | Introducing ML Systems Architecture | Sì (metà) | Architettura di produzione: contesto per task di serving/deploy |
| 7 | Data and Feature Management | Parziale | Feature store / gestione dati — coerente con la parte DVC del corso |
| 8 | Training Models with MLflow | **Sì** | Pattern di training tracciato, MLflow Projects, CI |
| 9 | Deployment and Inference with MLflow | **Sì** | `mlflow models serve`, Docker, REST inference: i comandi base sono invariati |
| 10 | Scaling Up Your ML Workflow | Opzionale | Tracking server scalato, artifact store remoti (S3) — rilevante se un task usa SeaweedFS/S3 come artifact store |
| 11 | Performance Monitoring | Opzionale | Drift/monitoring: utile solo se il corso lo copre (MLflow 3 ha MLflow Evaluate/monitoring nuovo) |
| 12 | Advanced Topics with MLflow | Opzionale | Topics avanzati: Projects, scoring server, integrazioni |

**Priorità di studio per le prove** (se il tempo è poco):

1. Libro B — Cap. 1, 4, 5, 8, 9
2. Libro A — Cap. 3, 4
3. Questa guida (`changes-mlflow.md`) come tavola di traduzione continua

---

## 10. Checklist rapida prima di ogni task MLflow

- [ ] `pip show mlflow` → versione? Se < 3, `pip install -U "mlflow>=3.1"`
- [ ] Backend store: `sqlite:///mlflow.db` se serve registry persistente
- [ ] Server: `mlflow server` (mai `mlflow ui`)
- [ ] `log_model` → `name=` non `artifact_path=`
- [ ] URI modello: `models:/<nome>/<stage|versione|@alias>`
- [ ] Tag git automatici non esistono più → `mlflow.set_tag` manuale se servono
- [ ] In dubbio: `mlflow <cmd> --help` o https://mlflow.org/docs/latest/
