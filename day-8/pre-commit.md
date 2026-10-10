# pre-commit — Guida introduttiva

## 1. Cos'è pre-commit?

`pre-commit` è un **framework** che gestisce ed esegue automaticamente dei controlli
(*hook*) sul tuo codice **prima che venga creato un commit Git**.

L'obiettivo è garantire qualità e coerenza del codice: formattazione, linting,
controllo di file, ecc. Invece di controllare a mano che tutto sia a posto prima di
committare, lasci che sia il framework a farlo per te — e a bloccare il commit se
qualcosa non va.

Il concetto di base: Git permette di definire degli *hook* (script) che vengono
eseguiti in momenti specifici, ad esempio:

- `pre-commit` → prima di ogni commit
- `pre-push` → prima di ogni push
- `commit-msg` → per validare il messaggio di commit

Scrivere questi script a mano è scomodo e poco portabile. `pre-commit` ti permette di
**dichiararli in un file YAML** e li installa/gestisce per te, isolando ogni hook nel
proprio ambiente.

---

## 2. Come funziona (il flusso)

1. Nel repo c'è un file di configurazione: **`.pre-commit-config.yaml`** (nella root).
2. Con `pre-commit install` viene installato uno script in `.git/hooks/pre-commit`.
3. Da quel momento, **ogni `git commit`** esegue gli hook **sui file in stage** (staged).
4. Se un hook **fallisce**, il commit viene **bloccato**.
5. Se un hook **modifica** dei file (auto-fix, es. `trailing-whitespace`, `black`),
   devi ri-aggiungerli con `git add` e riprovare il commit.
6. Gli hook girano in **ambienti isolati** (creati da pre-commit nella sua cache),
   quindi non "sporcano" il tuo ambiente Python/node/ecc.

> Nota: `pre-commit` non è un linter né un formatter. È il *gestore* che scarica,
> installa ed esegue altri tool (ruff, black, eslint, ...).

---

## 3. Il file `.pre-commit-config.yaml`

Struttura minima:

```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v6.0.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
```

Le chiavi:

| Chiave  | Significato |
|---------|-------------|
| `repo`  | URL del repository Git che contiene le definizioni degli hook |
| `rev`   | revisione da usare: **tag**, SHA di commit (o branch). Va *pinnata* |
| `hooks` | lista degli hook da eseguire da quel repository |

Per ogni hook:

| Chiave    | Significato |
|-----------|-------------|
| `id`      | **obbligatorio** — nome dell'hook (es. `trailing-whitespace`) |
| `name`    | nome mostrato (override) |
| `args`    | argomenti extra passati al tool (es. `["--fix"]`) |
| `files` / `exclude` | regex per includere/escludere file |
| `types`   | filtra per tipo di file (es. `[python]`) |
| `stages`  | in quali fasi gira (es. `[pre-commit, pre-push]`) |

Esistono anche gli **hook locali** (definiti nel repo stesso):

```yaml
  - repo: local
    hooks:
      - id: my-check
        name: my-check
        entry: ./scripts/check.sh
        language: system
        pass_filenames: false
```

### Perché `rev` è importante

`rev` **pinna** una versione precisa. Senza `rev` la configurazione è **invalida**
(pre-commit solleva `Missing required key: rev`). Le *branch* e i *tag mobili* non
sono supportati perché non sono riproducibili: due sviluppatori potrebbero eseguire
versioni diverse dello stesso hook.

---

## 4. Comandi principali

| Comando | Cosa fa |
|---------|---------|
| `pre-commit install` | installa l'hook git `pre-commit` |
| `pre-commit install --hook-type pre-push` | installa l'hook per il push |
| `pre-commit run` | esegue gli hook sui file in **stage** |
| `pre-commit run --all-files` | esegue gli hook su **tutti i file tracciati** |
| `pre-commit run <hook-id>` | esegue **un solo** hook |
| `pre-commit run --all-files --show-diff-on-failure` | mostra il diff quando fallisce |
| `pre-commit autoupdate` | aggiorna i `rev` all'**ultima release** di ogni repo |
| `pre-commit clean` | pulisce la cache degli ambienti |
| `pre-commit uninstall` | rimuove gli hook git |
| `pre-commit validate-config` | valida il file di configurazione |

`pre-commit autoupdate` è il modo standard per **scoprire le versioni correnti** senza
cercarle a mano: interroga ogni repository e riscrive i `rev`.

---

## 5. Esempio pratico completo

```yaml
repos:
  # 1) hook generici di manutenzione file
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v6.0.0
    hooks:
      - id: trailing-whitespace      # rimuove spazi a fine riga
      - id: end-of-file-fixer        # assicura il newline finale
      - id: check-yaml               # valida la sintassi YAML

  # 2) linter/formatter Python: ruff
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.16.10
    hooks:
      - id: ruff

  # 3) formatter Python: black
  - repo: https://github.com/psf/black-pre-commit-mirror
    rev: 26.10.0
    hooks:
      - id: black
```

Flusso di lavoro tipico:

```bash
pre-commit install          # una volta sola, registra l'hook in git
pre-commit run --all-files  # prova su tutto il repo
git add .                   # se gli hook hanno modificato file
git commit -m "..."         # ora il commit esegue gli hook
```

---

## 6. Hook comuni di `pre-commit/pre-commit-hooks`

- `trailing-whitespace` — rimuove gli spazi a fine riga
- `end-of-file-fixer` — garantisce il newline finale
- `check-yaml` / `check-json` / `check-toml` — validano la sintassi
- `check-added-large-files` — blocca file troppo grandi
- `check-merge-conflict` — intercetta marker di conflitto
- `detect-private-key` — blocca chiavi private
- `mixed-line-ending` — normalizza CRLF/LF

---

## 7. Errori comuni (e come riconoscerli)

| Errore | Correzione |
|--------|------------|
| `id: check_yaml` (underscore) | l'id corretto è `check-yaml` (con **hyphen**) |
| `repo: charliermarsh/ruff-pre-commit` | il repo ufficiale è ora `astral-sh/ruff-pre-commit` |
| `id: ruff-lint` | l'id corretto è `ruff` |
| `Missing required key: rev` | manca `rev` su una `repo` → aggiungilo |
| `appears to be a mutable reference` | stai usando branch/tag mobile → `pre-commit autoupdate` |
| hook che falliscono sempre | i file non sono formattati/linti → lancia gli hook e applica i fix |

---

## 8. Applicazione al laboratorio (KodeKloud)

Configurazione **corretta** da ottenere (dopo `pre-commit autoupdate`):

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

Procedura:

```bash
cd /root/code/fraud-detection
pre-commit run --all-files      # verifica che tutto passi "clean"
pre-commit install              # registra l'hook in git
git add .pre-commit-config.yaml process.py
git commit -m "fix: pre-commit config"
```
