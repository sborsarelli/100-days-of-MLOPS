# Day 11 — Migrating a Git-Tracked Dataset to DVC

## Overview

Part of the *100 Days of MLOps* challenge. This exercise takes a dataset that was committed
directly to [Git](https://git-scm.com/) and hands it over to
[DVC](https://dvc.org/) (Data Version Control), which is the team standard for every file
under `data/`. The write-up documents the initial state, the exact commands used, the
verification of the four acceptance criteria, and pointers for deeper study.

The scenario: a teammate added the fraud-detection transactions dataset to the
xFusionCorp Industries repository and committed it straight to Git. The project at
`/root/code/fraud-detection/` already has DVC initialised, so the dataset must be moved
from Git tracking to DVC tracking — Git keeps the pointer, DVC keeps the data.

## Task

The end state must satisfy all of the following:

- Git no longer tracks the dataset, but the file remains on disk.
- The dataset is tracked by DVC instead: a `.dvc` pointer file exists and
  `data/raw/.gitignore` excludes the dataset itself.
- The new `.dvc` pointer and `.gitignore` are recorded in a Git commit with the message
  `Track transactions dataset with DVC`.
- Once tracking is moved to DVC, the **DVC TRACKED** section in the Explorer panel lists
  the dataset, confirming the extension recognises it as a DVC-managed file.

## Background: Git tracking vs DVC tracking

Git and DVC both "track" a file, but they mean opposite things. Git stores the file's
**content** in its object database; DVC stores the content in its own cache (and,
optionally, a remote) and commits only a small **metafile** that references it by hash.
That is why a single path cannot be tracked by both at once — `dvc add` treats a
Git-tracked path as a hard error.

### What `dvc add` actually does

`dvc add <path>` performs three side effects:

1. copies the file content into the local cache (`.dvc/cache/`), keyed by hash;
2. writes a **pointer file** `<path>.dvc` containing the output path and its hash;
3. appends an entry to a **`.gitignore`** next to the data, so Git skips the file.

The generated pointer for this exercise is small enough to read directly:

```yaml
outs:
- md5: <hash>
  size: <bytes>
  hash: md5
  path: transactions.csv
```

The pointer and the `.gitignore` are the two artefacts that get committed; the data itself
is not.

### Why a `.gitignore` rule cannot un-track a file

`.gitignore` only affects **untracked** files. Once a path is in Git's index, the ignore
rule is silently ignored for it — Git keeps versioning the file. GitHub's own
documentation states the requirement plainly: *"If you want to ignore a file that is
already checked in, you must untrack the file before you add a rule to ignore it."*

This is exactly the trap in this exercise: running `dvc add` first fails, and even if the
`.gitignore` entry were written by hand, Git would keep tracking the dataset and the
acceptance criteria would not be met.

### The DVC guard-rail

DVC enforces the one-owner rule. In `dvc/output.py`, `Output.ignore()` checks the SCM
before writing the ignore entry:

```python
def ignore(self) -> None:
    if not self.use_scm_ignore:
        return

    assert self.repo
    if self.repo.scm.is_tracked(self.fspath):
        raise OutputAlreadyTrackedError(self)

    self.repo.scm_context.ignore(self.fspath)
```

`OutputAlreadyTrackedError` prints the fix for you:

```console
ERROR: output 'data/raw/transactions.csv' is already tracked by SCM (e.g. Git).
You can remove it from Git, then add to DVC.
To stop tracking from Git:
    git rm -r --cached 'data/raw/transactions.csv'
    git commit -m "stop tracking data/raw/transactions.csv"
```

So the error is not an obstacle to work around — it *is* the instruction to un-track first.

### `git rm --cached`: untrack, do not delete

The Git manual is explicit about the flag:

> `--cached` — Use this option to unstage and remove paths only from the index. Working
> tree files, whether modified or not, will be left alone.

Without `--cached`, `git rm` deletes the file from the working tree as well, which would
break the first acceptance criterion (the file must remain on disk for DVC to cache it).

Relevant commands:

| Command | Effect |
| --- | --- |
| `git rm --cached <path>` | remove `<path>` from the Git index only; the file stays on disk |
| `dvc add <path>` | cache the data, write `<path>.dvc`, add the path to `.gitignore` |
| `git ls-files <dir>` | list the paths Git currently tracks (proof of un-tracking) |
| `git commit --amend -m "…"` | rewrite the message of the last, unpushed commit |
| `dvc status` | compare tracked data against the cache |

## Initial state

The dataset is committed to Git; DVC is already initialised (`.dvc/` exists) but owns
nothing yet:

```console
$ cd /root/code/fraud-detection/
$ git status
On branch main
nothing to commit, working tree clean

$ git ls-files data/raw/
data/raw/transactions.csv
```

| Requirement | Initial state |
| --- | --- |
| Git does not track the dataset | violated — the csv is tracked |
| `.dvc` pointer exists | missing |
| `data/raw/.gitignore` excludes the dataset | missing |
| commit `Track transactions dataset with DVC` | missing |

## Solution

### 1. Untrack the dataset from Git

```bash
cd /root/code/fraud-detection/
git rm --cached data/raw/transactions.csv
```

```console
$ git rm --cached data/raw/transactions.csv
rm 'data/raw/transactions.csv'
```

The shell prompt turns into `main [✘?]`: a deletion is staged and the dataset is now
untracked-but-present. `--cached` is what guarantees the file is still on disk.

### 2. Hand the dataset over to DVC

```bash
dvc add data/raw/transactions.csv
```

```console
$ dvc add data/raw/transactions.csv
100% Adding...|███████████████████████|1/1 [00:00, 50.04file/s]

To track the changes with git, run:

        git add data/raw/.gitignore data/raw/transactions.csv.dvc

To enable auto staging, run:

        dvc config core.autostage true
```

This is the step that would have failed with `OutputAlreadyTrackedError` had step 1 been
skipped. DVC now owns the data: the content is in `.dvc/cache/`, the pointer
`data/raw/transactions.csv.dvc` exists, and `data/raw/.gitignore` contains the entry
`/transactions.csv` (leading slash = relative to that directory).

### 3. Commit the pointer

`dvc add` does **not** stage anything itself, so both new files must be added explicitly:

```bash
git add data/raw/transactions.csv.dvc data/raw/.gitignore
git commit -m "Track transactions dataset with DVC"
```

### 4. Fix the commit-message typo

The first attempt was committed as `rack transactions dataset with DVC` (a missing `t`).
Since the commit had not been pushed and the working tree was clean, amending the message
is safe and does not touch the tree:

```console
$ git commit --amend -m "Track transactions dataset with DVC"
[main 739674d] Track transactions dataset with DVC
 Date: Sun Oct 11 12:53:10 2026 -0400
 3 files changed, 6 insertions(+), 11 deletions(-)
 create mode 100644 data/raw/.gitignore
 delete mode 100644 data/raw/transactions.csv
 create mode 100644 data/raw/transactions.csv.dvc
```

Note that the hash changed (`515e922` → `739674d`): `--amend` rewrites the commit rather
than editing it in place.

## Verification

Run the following from `/root/code/fraud-detection`.

**1. Git no longer tracks the dataset, but the file is still on disk**

```bash
git ls-files data/raw/          # expected: .gitignore and transactions.csv.dvc only
ls -lh data/raw/transactions.csv
```

**2. DVC owns the dataset**

```bash
cat data/raw/transactions.csv.dvc   # expected: outs: path: transactions.csv + md5
cat data/raw/.gitignore             # expected: /transactions.csv
dvc status                          # expected: Data and pipelines are up to date.
```

**3. The commit message is exactly right**

```bash
git log -1 --pretty=%s              # expected: Track transactions dataset with DVC
git show --stat HEAD                # expected: the 3 files above
```

Recorded end state of the session:

```console
$ git log --oneline -1
739674d (HEAD -> main) Track transactions dataset with DVC

$ git ls-files data/raw/
data/raw/.gitignore
data/raw/transactions.csv.dvc

$ git log -1 --pretty=%s
Track transactions dataset with DVC
```

**4. The extension recognises it**

With the DVC extension open on the project, the **DVC TRACKED** section of the Explorer
lists `transactions.csv` (it is driven by the `.dvc` pointer file). If the panel does not
refresh immediately, reload the window.

The end state is reached when Git tracks only the pointer and the `.gitignore`, the data
file is still present on disk, `dvc status` reports everything up to date, and the commit
carries the exact message `Track transactions dataset with DVC`.

## Gotchas

### `dvc add` fails on a Git-tracked path — and that is by design

`ERROR: output '…' is already tracked by SCM (e.g. Git)` is not a bug to bypass with
`--force`. The same error also appears when *part* of a directory added with `dvc add` is
Git-tracked, so the fix is always the same: un-track from Git first, then `dvc add`.

### `--cached` is mandatory, not optional

Plain `git rm data/raw/transactions.csv` deletes the file from the working tree. The first
acceptance criterion requires the file to survive on disk — and `dvc add` needs it there
to populate the cache. Only `git rm --cached` removes the path from the index alone.

### `.gitignore` cannot un-track anything

Adding `/transactions.csv` to `.gitignore` while Git still tracks the file changes
nothing. Ignore rules apply to untracked paths only; tracked paths override them. That is
why the order is *un-track, then let DVC write the ignore entry* — never the reverse.

### `git rm --cached` refuses when the index is dirty

The Git manual notes that, with `--cached`, "the staged content has to match either the
tip of the branch or the file on disk". If the dataset were modified *and staged*, the
command would abort with *"has staged content different from both the file and the HEAD"*.
Unstage it first (`git restore --staged <path>`) or use `-f` deliberately.

### `dvc add` stages nothing

Unlike `dvc init`, which runs `git add` for you, `dvc add` only *prints* the command to
run. Forgetting to stage the `.dvc` pointer and the `.gitignore` leaves the repository
clean but the migration uncommitted. Setting `dvc config core.autostage true` automates
that staging step for future adds.

### `--amend` rewrites history

Amending the message changes the commit hash, so it is only safe while the commit is
unpushed. Also note that `--amend` folds **anything currently staged** into the commit —
if the tree is not clean, unstage first or the fix silently becomes a content change.

### The old blob is still in Git's history

`git rm --cached` stops tracking the dataset from HEAD onwards, but the blob committed
earlier still exists in the object database (the original commit becomes dangling). The
acceptance criteria only look at what Git tracks now, so this is expected. Purging the
blob from history requires `git filter-repo` or [BFG](https://rtyley.github.io/bfg-repo-cleaner/),
which rewrites every subsequent commit — never do it casually.

### `dvc add` keeps the data on disk and in the cache

DVC does not move or delete the working file: it stays where it was and a copy lands in
`.dvc/cache/` (and later on the remote, once `dvc push` is run). The workspace file is
relinked from the cache on `dvc checkout` after a `git checkout` of the pointer.

### Line wrapping in `git log --oneline` is not a broken message

The transcript shows the typo'd message as two lines:

```console
$ git log --oneline -1
515e922 (HEAD -> main) r
ack transactions dataset with DVC
```

That is terminal wrapping, not a newline inside the message. `git log -1 --pretty=%s`
prints the subject as a single line and is the reliable way to check it.

## Further reading

### Books

- **The Pragmatic Programmer for Machine Learning** — Marco Scutari & Mauro Malvestio
  (Chapman & Hall/CRC, 2023). **Open access** at <https://ppml.dev/>. Chapter 10,
  *"Tools for Developing Pipelines"*, introduces DVC as "GitOps for data" and covers data
  versioning and provenance — the conceptual frame for why a dataset should not live in
  Git in the first place.
- **Building Machine Learning Pipelines** — Hannes Hapke & Catherine Nelson (O'Reilly,
  2020). Chapter 3, *"Data Ingestion"*, has a *"Versioning Datasets"* section on the
  data-versioning problem and the tools that address it.
  <https://www.oreilly.com/library/view/building-machine-learning/9781492053187/>
- **Pro Git** — Scott Chacon & Ben Straub (Apress, 2014), free at
  <https://git-scm.com/book/en/v2>. Chapter 2, *"Git Basics"*, covers both sides of this
  exercise: *"Ignoring Files"* (`.gitignore` semantics) and *"Removing Files"*
  (`git rm --cached`).
  - *Note:* as with the other days, no book is devoted **exclusively** to the Git↔DVC
    migration; the chapters above are the closest coverage inside broader texts.

### Official documentation

- `dvc add` command reference — <https://doc.dvc.org/command-reference/add> — documents the
  `.dvc` pointer, the automatic `.gitignore` entry, and `core.autostage`.
- *How to Stop Tracking Data* — <https://doc.dvc.org/user-guide/how-to/stop-tracking-data> —
  the inverse operation (`dvc remove` + `dvc gc -w`), useful to understand what "DVC owns
  this file" implies.
- `.dvc` files — <https://doc.dvc.org/user-guide/project-structure/dvc-files> — the format
  of the pointer file created here.
- Tutorial: *Data and Model Versioning* —
  <https://doc.dvc.org/example-scenarios/versioning-data-and-models/tutorial> — the canonical
  hands-on walkthrough of `dvc add` and remotes.
- `git rm` — <https://git-scm.com/docs/git-rm> — the authoritative description of
  `--cached` and of the up-to-date check that can force `-f`.
- `gitignore` — <https://git-scm.com/docs/gitignore> — what ignore rules do and do not
  apply to.
- GitHub Docs, *Ignoring files* —
  <https://docs.github.com/en/get-started/getting-started-with-git/ignoring-files> — states
  the rule used here in one line: untrack first, then ignore.
- VS Code DVC extension —
  <https://marketplace.visualstudio.com/items?itemName=lakeFS.lakefs-dvc> — the extension
  that renders the **DVC TRACKED** panel in the acceptance criteria (see also
  <https://doc.dvc.org/install/ide-plugins>).
- Source repository — <https://github.com/treeverse/dvc> (formerly `iterative/dvc`); the
  guard-rail lives in `dvc/output.py` (`OutputAlreadyTrackedError`).

### Blog articles and discussions

- DVC Discuss, *"Output '' is already tracked by SCM (e.g. Git)"* —
  <https://discuss.dvc.org/t/output-is-already-tracked-by-scm-e-g-git/419> — the
  maintainers' explanation of the error and the recommended `git rm --cached` fix.
- DVC issue #3678, *add/run: improve "already tracked" message* —
  <https://github.com/iterative/dvc/issues/3678> — the discussion that added the Git
  suggestions to the error message.
- DVC issue #5728, *add: fail with output is already tracked by SCM but those tracked file
  is ignored by dvc* — <https://github.com/iterative/dvc/issues/5728> — why mixing Git- and
  DVC-tracked files inside one directory is intentionally rejected.

### Video tutorials

- **DVCorg — "Sharing Data and Models with DVC (Hands-On Data Science Tutorial!)"** —
  <https://www.youtube.com/watch?v=EE7Gk84OZY8> — a beginner-oriented walkthrough of
  versioning data with Git + DVC, i.e. the workflow this day migrates into.
- **DVCorg — "Reaching the next level of ML experiment tracking with the DVC 3.0 Stack"** —
  <https://www.youtube.com/watch?v=u-URI5Lvc-g> — demonstrates the VS Code DVC extension
  and its **DVC TRACKED** panel.

## Repository notes

- This directory documents the exercise; the project itself lives at
  `/root/code/fraud-detection/` and is not committed here.
- `task.txt` is the exercise brief and is kept local only (see `.gitignore`).
- The transcript above is the actual session, including the `git coomit` typo and the
  `--amend` correction; the final commit hash is `739674d`.
