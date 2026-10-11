# Day 10 — Initialising DVC in the Fraud-Detection Repository

## Overview

Part of the *100 Days of MLOps* challenge. This exercise initialises
[DVC](https://dvc.org/) (Data Version Control) inside the existing fraud-detection Git
repository, so that datasets and model files can be versioned independently from the
codebase, and records that initialisation in Git.

The scenario: the xFusionCorp Industries ML team is adopting DVC to keep large datasets
and trained models out of Git while still versioning them. A Git repository already exists
at `/root/code/fraud-detection/` with an initial commit; DVC must be initialised there and
the files it produces committed.

## Task

The end state must satisfy the following:

- DVC is initialised inside the repository, so the standard `.dvc/` control directory and
  `.dvcignore` file exist alongside the existing Git working tree.
- Every file DVC produces during initialisation is recorded in a **new Git commit** with
  the message `Initialize DVC`.
- Once initialisation is complete, the DVC extension detects the new `.dvc/` directory and
  surfaces the **DVC TRACKED** section in the Explorer panel, together with a DVC indicator
  in the bottom status bar.

## Background: what DVC is

DVC (Data Version Control) is an open-source **version-control layer for data and models**
that works *on top of* Git. Git is excellent at tracking small text files (code) but
unsuitable for large binaries (datasets, model weights): they bloat the repository and
every change stores a full copy. DVC solves this by storing the actual data in a local
**cache** (and, optionally, a **remote**) and committing only small **metafiles** to Git.
The payoff: a `git checkout` restores the exact data version associated with a commit.

Key ideas:

- **Metadata, not data, in Git** — DVC writes a small `.dvc` file (or a `dvc.yaml`) holding
  a content hash; the data itself lives in the cache.
- **Content-addressable cache** — data is stored under `.dvc/cache/` keyed by hash (MD5 by
  default), so identical content is stored once.
- **Remotes** — `dvc push` / `dvc pull` synchronise the cache with remote storage (S3, GCS,
  SSH, …).
- **Pipelines and experiments** — `dvc.yaml` defines reproducible stages; DVC can reproduce
  and compare experiments.
- **Git-native** — DVC relies on a Git repository (`.dvc/` sits next to `.git/`), so
  branches, tags and commits version data along with code.

### The `.dvc/` control directory

`dvc init` creates the hidden `.dvc/` directory, which holds DVC's internal state:

| Path | Role |
| --- | --- |
| `.dvc/config` | default project configuration (committed to Git) |
| `.dvc/config.local` | optional, Git-ignored local overrides (secrets, private remotes) |
| `.dvc/.gitignore` | keeps the cache and temp files out of Git (`/config.local`, `/tmp`, `/cache`) |
| `.dvc/tmp/` | temporary files and lock files (not committed) |
| `.dvc/cache/` | the content-addressable cache — created lazily on the first `dvc add`, not at init (not committed) |

Alongside it, `dvc init` creates a **`.dvcignore`** file at the project root — a
`.gitignore`-style list of paths DVC should skip while traversing the project. It is
optional and empty right after initialisation. Both `.dvc/config` and `.dvcignore` are
staged by `dvc init` and are meant to be committed.

Common commands (for context):

| Command | Effect |
| --- | --- |
| `dvc init` | initialise DVC in the current Git repo; creates and stages `.dvc/` + `.dvcignore` |
| `dvc add <path>` | start tracking a file/directory; writes `<path>.dvc` and adds the data to `.gitignore` |
| `dvc status` | show the state of tracked data relative to the cache and Git |
| `dvc push` / `dvc pull` | upload/download tracked data to/from the configured remote |
| `dvc remote add` | configure a storage remote |

## Initial state

A plain Git repository — no DVC yet:

```console
$ cd /root/code/fraud-detection/
$ ls -la
total 32
drwxr-xr-x 6 root root 4096 Oct 11 12:25 .
drwxr-xr-x 3 root root 4096 Oct 11 12:25 ..
drwxr-xr-x 8 root root 4096 Oct 11 12:25 .git
-rw-r--r-- 1 root root   13 Oct 11 12:25 .gitignore
-rw-r--r-- 1 root root   18 Oct 11 12:25 README.md
drwxr-xr-x 3 root root 4096 Oct 11 12:25 data
drwxr-xr-x 2 root root 4096 Oct 11 12:25 models
drwxr-xr-x 2 root root 4096 Oct 11 12:25 src
```

No `.dvc/` directory and no `.dvcignore` file exist.

## Solution

### 1. Initialise DVC inside the repository

```bash
cd /root/code/fraud-detection/
dvc init
```

```console
$ dvc init
Initialized DVC repository.

You can now commit the changes to git.

+---------------------------------------------------------------------+
|                                                                     |
|        DVC has enabled anonymous aggregate usage analytics.         |
|     Read the analytics documentation (and how to opt-out) here:     |
|             <https://dvc.org/doc/user-guide/analytics>              |
|                                                                     |
+---------------------------------------------------------------------+

What's next?
------------
- Check out the documentation: <https://dvc.org/doc>
- Get help and share ideas: <https://dvc.org/chat>
- Star us on GitHub: <https://github.com/treeverse/dvc>
```

`dvc init` requires a Git repository (it walks up the tree until it finds `.git/`). It
creates `.dvc/` and `.dvcignore`, and **stages them with `git add` automatically** — hence
the "You can now commit the changes to git" message.

### 2. Commit the initialisation

No manual `git add` is needed; `dvc init` already staged the files:

```console
$ git status
On branch master
Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
        new file:   .dvc/.gitignore
        new file:   .dvc/config
        new file:   .dvcignore
```

```console
$ git commit -m 'Initialize DVC'
[master 38a41c5] Initialize DVC
 3 files changed, 6 insertions(+)
 create mode 100644 .dvc/.gitignore
 create mode 100644 .dvc/config
 create mode 100644 .dvcignore
```

The commit message matches the brief exactly (`Initialize DVC`), and the commit contains
exactly the three files DVC produced during initialisation.

## Verification

Run the following from the project directory (`/root/code/fraud-detection`).

**1. The DVC control files exist**

```bash
ls -a .dvc                 # expected: .gitignore  config  tmp
ls -a | grep .dvcignore    # expected: .dvcignore
```

`.dvc/cache/` appears only once data is first tracked with `dvc add`; it is absent right
after initialisation.

**2. The initialisation is committed**

```bash
git log --oneline -1              # expected: 38a41c5 Initialize DVC
git show --stat --oneline HEAD    # expected: the three files above
```

**3. DVC is active in the project**

```bash
dvc version
dvc status
```

Expected final output:

```console
$ git log --oneline -1
38a41c5 Initialize DVC

$ dvc status
Data and pipelines are up to date.
```

The end state is reached when `.dvc/` and `.dvcignore` exist, the `Initialize DVC` commit
records exactly the files DVC staged, and the VS Code DVC extension shows the **DVC TRACKED**
section in the Explorer panel with the DVC status-bar indicator.

## Gotchas

### `dvc init` stages the files for you

There is no need to run `git add`: `dvc init` calls `git add` on `.dvc/` and `.dvcignore`
itself. The `[+]` in the shell prompt and the "Changes to be committed" block are the
proof — committing straight away is correct.

### `.dvc/tmp` and `.dvc/cache` never appear in the commit

`.dvc/tmp` is created by initialisation and `.dvc/cache` is created once data is first
tracked, but both are covered by `.dvc/.gitignore` (`/config.local`, `/tmp`, `/cache`).
Only `.dvc/config`, `.dvc/.gitignore` and `.dvcignore` are tracked — this is expected, not a
missing file.

### `.dvcignore` is empty right after init

It is an *optional* exclusion list (a `.gitignore` for DVC's directory traversal), not an
inventory of tracked files. An empty file is normal.

### Do not "fix" the analytics notice by editing `.dvc/config`

`dvc init` enables anonymous usage analytics and prints how to opt out. Opting out
(`dvc config core.analytics false`) rewrites `.dvc/config` and changes the committed
artefact — it is **not** required by the task, so leave the default in place.

### `dvc init` needs a Git repository

Outside a Git repo it aborts (`you are not inside of a git repository`). `dvc init --no-scm`
initialises DVC *without* Git, but that is not what this exercise asks for.

### `treeverse/dvc` in the output is current, not a typo

Recent DVC releases print `https://github.com/treeverse/dvc`: the project was acquired by
lakeFS/Treeverse (November 2025) and the repository moved from `iterative/dvc` to
`treeverse/dvc`. The docs remain at `dvc.org/doc` (also reachable at `doc.dvc.org`).

### Re-running `dvc init`

A second run prints `DVC is already initialized`. Use `dvc init --force` to rebuild a
corrupted `.dvc/` — but that discards the local cache, so only do it deliberately.

## Further reading

### Books

- **The Pragmatic Programmer for Machine Learning** — Marco Scutari & Mauro Malvestio
  (Chapman & Hall/CRC, 2023). **Open access** at <https://ppml.dev/>. Chapter 10,
  *"Tools for Developing Pipelines"*, is the closest book-length treatment of DVC: it
  introduces DVC as "GitOps for data" and covers data versioning, pipelines and provenance,
  contrasting it with MLflow.
- **Building Machine Learning Pipelines** — Hannes Hapke & Catherine Nelson (O'Reilly,
  2020). Chapter 3, *"Data Ingestion"*, includes a *"Versioning Datasets"* section on the
  data-versioning problem and the tools that address it (the book is TFX-centric, so DVC
  appears as context rather than a hands-on walkthrough).
  <https://www.oreilly.com/library/view/building-machine-learning/9781492053187/>
  - *Note:* there is no book devoted **exclusively** to DVC; the two chapters above are the
    closest coverage inside broader MLOps / data-engineering texts.

### Official documentation

- **Online tutorial — Get Started with DVC**: <https://doc.dvc.org/start> (redirects from
  <https://dvc.org/doc/start>) — the canonical hands-on walkthrough: initialising a project,
  tracking data with `dvc add`, and setting up remotes. It reproduces the exact
  `Initialize DVC` commit this exercise performs.
- `dvc init` command reference — <https://doc.dvc.org/command-reference/init> — documents
  the contents of `.dvc/` and the automatic `git add`.
- Project structure — <https://doc.dvc.org/user-guide/project-structure> — what `dvc.yaml`,
  `.dvc` files and `.dvcignore` are.
- `.dvcignore` files — <https://doc.dvc.org/user-guide/project-structure/dvcignore-files>.
- Internal files — <https://doc.dvc.org/user-guide/project-structure/internal-files> — the
  full inventory of `.dvc/`.
- Tutorial: Data and Model Versioning —
  <https://doc.dvc.org/example-scenarios/versioning-data-and-models/tutorial>.
- Source repository — <https://github.com/treeverse/dvc> (formerly `iterative/dvc`).

### Video tutorials

- **DVCorg — "Sharing Data and Models with DVC (Hands-On Data Science Tutorial!)"** —
  <https://www.youtube.com/watch?v=EE7Gk84OZY8> — a gentle, beginner-oriented walkthrough of
  versioning data and models with Git + DVC.
- **DVCorg — "Reaching the next level of ML experiment tracking with the DVC 3.0 Stack"** —
  <https://www.youtube.com/watch?v=u-URI5Lvc-g> — demonstrates the VS Code DVC extension,
  DVCLive and experiment tracking (the extension referenced in the acceptance criteria).

## Repository notes

- This directory documents the exercise; the project itself lives at
  `/root/code/fraud-detection/` and is not committed here.
- `task.txt` is the exercise brief and is kept local only (see `.gitignore`).
