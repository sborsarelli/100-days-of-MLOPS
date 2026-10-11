# Day 13 — Fixing a DVC Pull from an S3-Compatible Remote (SeaweedFS)

## Overview

Part of the *100 Days of MLOps* challenge. Day 12 fixed a **`dvc push`** that could not
reach the team's S3-compatible store; this day fixes the mirror image: a **`dvc pull`** that
fails on a fresh clone because the credentials are missing from the DVC configuration.

The xFusionCorp Industries ML team keeps [DVC](https://dvc.org/)-tracked data in a shared
**SeaweedFS** S3-compatible object store instead of in Git. A new teammate cloned the
`fraud-detection` project onto a fresh machine: the repository, the `.dvc` pointer file and
the remote definition all came along, but the dataset did not — it is absent both from disk
and from the local DVC cache, and `dvc pull` aborts before it can fetch anything.

The write-up documents the failure, the single missing configuration key behind it, the
exact commands used, the verification of both acceptance criteria, a reusable tutorial for
diagnosing a failing `dvc pull` against any S3-compatible bucket, and pointers for deeper
study.

## Task

The environment:

| Fact | Value |
| --- | --- |
| Project | `/root/code/fraud-detection/` (cloned; DVC initialised; `data/raw/transactions.csv.dvc` present) |
| Dataset state | `data/raw/transactions.csv` missing from disk **and** from the local DVC cache |
| Object store | SeaweedFS, S3 endpoint `http://localhost:8333` |
| Filer UI | forwarded port `8888`, buckets under `/buckets/dvc-storage/` |
| Credentials | `weedadmin` / `weedadmin123` |
| Bucket | `dvc-storage` (already created, dataset already pushed) |

Acceptance criteria:

- The `s3` remote in `.dvc/config` reaches SeaweedFS with the access key
  (`access_key_id`) `weedadmin` and the secret key (`secret_access_key`) `weedadmin123`.
- After the pull, `data/raw/transactions.csv` is present on disk and its content matches the
  hash recorded in the `.dvc` pointer.

## Background: how `dvc pull` works and where credentials come from

### `dvc pull` = `dvc fetch` + `dvc checkout`

A pull is two operations glued together:

1. **`dvc fetch`** — download the missing objects from the remote into the local cache
   (`.dvc/cache/`);
2. **`dvc checkout`** — copy the objects from the cache into the workspace.

This is why the successful run reports *two* counts — `1 file fetched and 1 file added`:
one object moved remote → cache, one file materialised cache → workspace (the `A` prefix on
`data/raw/transactions.csv`). On a fresh clone the local cache is empty, so the fetch has
real work to do; on a warm cache only the checkout runs.

The remote itself is not a filesystem mount — DVC lists and reads objects through the S3
API, which is the point at which credentials are needed.

### The credential resolution chain

When DVC talks to an S3 remote it hands the explicit `access_key_id` / `secret_access_key`
options to its S3 client (`s3fs`, on top of `boto3`). If those keys are **absent**, the
client falls back to the standard AWS credential chain, in order:

1. inline DVC options — `access_key_id` / `secret_access_key` in the DVC config;
2. environment variables — `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` / `AWS_SESSION_TOKEN`;
3. shared credentials file — `~/.aws/credentials` (optionally via a named `profile`);
4. instance/container role metadata (EC2, ECS, IRSA);
5. otherwise: **fail** with `Unable to locate credentials`.

The lab's failure is the *last* case: none of the sources above exist on the fresh clone,
and the two inline options are precisely what is missing. That is why the message is a
credential error, not a connection error.

### Where DVC reads remote options from

The remote definition is merged from four config levels, each shadowing the previous one:

| Level | File | Committed? |
| --- | --- | --- |
| system | `/etc/dvc/config` | no |
| global | `~/.config/dvc/config` | no |
| project | `.dvc/config` | **yes** — this is the file the lab ships |
| local | `.dvc/config.local` | no (git-ignored by `.dvc/.gitignore`) |

`dvc remote modify` writes to the **project** level (`.dvc/config`) by default; adding
`--local` writes to the git-ignored `.dvc/config.local` instead. Both are *read* by
`dvc pull`, so either would fix this lab — the acceptance criterion asks specifically for
`.dvc/config`.

### The remote layout: `files/md5/...`

DVC 3.x hashes content with **md5** (`hash: md5` in the pointer) and stores objects under a
two-level sharding scheme:

```
files/md5/<first 2 hex chars>/<remaining 30 hex chars>
```

So a `transactions.csv` with hash `555f037ee350464f52122d087f28e857` lives at
`files/md5/55/5f037ee350464f52122d087f28e857`. The error message names this path as
`dvc-storage/files/md5` — that is **bucket + prefix**, the location DVC tried to list, not a
path on the local disk.

### What "Unable to locate credentials" actually means

The string is `botocore.exceptions.NoCredentialsError` surfacing through DVC. It is raised
*before* any HTTP request to SeaweedFS: the client refuses to sign a request with no key.
The wording is literal — the endpoint and the bucket were never contacted.

## Initial state

The clone is complete and the remote resolves, but the credentials are absent and the
dataset is not on disk:

```console
$ cd /root/code/fraud-detection/
$ dvc pull
Collecting                           |1.00 [00:00,  782entry/s]
ERROR: failed to connect to s3 (dvc-storage/files/md5) - Unable to locate credentials
Fetching
ERROR: failed to pull data from the cloud - 1 files failed to download

$ dvc config --list
remote.s3.url=s3://dvc-storage
remote.s3.endpointurl=http://localhost:8333
core.remote=s3
```

| Requirement | Initial state |
| --- | --- |
| `s3` remote carries `access_key_id` / `secret_access_key` | violated — both keys missing |
| URL is `s3://dvc-storage` | satisfied |
| Endpoint is `http://localhost:8333` | satisfied |
| A default remote is set | satisfied (`core.remote=s3`) |
| Dataset present on disk | violated — missing |
| Dataset present in the local cache | violated — missing |

A single defect: the two credential keys. The URL, the endpoint and the default remote — the
three things Day 12 had to fix on the *push* side — are already correct here, which is why
the failure surfaces at authentication rather than at config resolution.

## Solution

### 1. Reproduce the failure

```bash
cd /root/code/fraud-detection/
dvc pull
```

The error `Unable to locate credentials` already points at credentials, not connectivity.

### 2. Inspect the remote configuration

```console
$ dvc config --list
remote.s3.url=s3://dvc-storage
remote.s3.endpointurl=http://localhost:8333
core.remote=s3
```

The absence of `remote.s3.access_key_id` / `remote.s3.secret_access_key` is the root cause.
The endpoint and URL match the brief, so nothing there needs changing.

### 3. Add the missing credentials

```bash
dvc remote modify s3 access_key_id weedadmin
dvc remote modify s3 secret_access_key weedadmin123
```

With no scope flag, both keys are written to the **project** `.dvc/config` — exactly what
the acceptance criterion asks for. (For a real repository the secrets would go to
`.dvc/config.local` with `--local`; see *Gotchas*.)

### 4. Confirm the resulting configuration

```console
$ cat .dvc/config
[core]
    remote = s3
['remote "s3"']
    url = s3://dvc-storage
    endpointurl = http://localhost:8333
    access_key_id = weedadmin
    secret_access_key = weedadmin123
```

## Verification

Run the following from `/root/code/fraud-detection`.

**1. The pull succeeds**

```console
$ dvc pull
Collecting                           |0.00 [00:00,    ?entry/s]
Fetching
Building workspace index             |2.00 [00:00,  709entry/s]
Comparing indexes                   |4.00 [00:00, 3.12kentry/s]
Applying changes                     |1.00 [00:00, 1.11kfile/s]
A       data/raw/transactions.csv
1 file fetched and 1 file added
```

`fetched` = downloaded into the cache; `added` = checked out into the workspace. One object
is expected: the dataset is a single file (a `.dir` object would only appear for a directory
output).

**2. The dataset is on disk**

```console
$ ls -la data/raw/transactions.csv
-rw-r--r-- 1 root root 446 Oct 11 16:40 data/raw/transactions.csv
```

**3. The content matches the hash recorded in the pointer**

This is the direct check of acceptance criterion #2 — recompute the hash and compare it with
the `.dvc` pointer:

```console
$ cat data/raw/transactions.csv.dvc
outs:
- md5: 555f037ee350464f52122d087f28e857
  size: 446
  hash: md5
  path: transactions.csv

$ md5sum data/raw/transactions.csv
555f037ee350464f52122d087f28e857  data/raw/transactions.csv
```

The recomputed `555f037ee350464f52122d087f28e857` is byte-for-byte the pointer's `md5`, and
the recorded `size` (446) matches the on-disk size. DVC 3.x stores an uncompressed copy, so
the size must match exactly.

**4. The object is in the bucket under `files/md5/`**

```console
$ AWS_ACCESS_KEY_ID=weedadmin AWS_SECRET_ACCESS_KEY=weedadmin123 \
    aws --endpoint-url http://localhost:8333 s3 ls s3://dvc-storage/files/md5/ --recursive
2026-10-11 16:27:12        446 files/md5/55/5f037ee350464f52122d087f28e857
```

`55` + `5f037ee350464f52122d087f28e857` recomposes the pointer hash exactly, and the object
size (446) matches. The same path is visible in the Filer UI under
`/buckets/dvc-storage/files/md5/`. Note that `--endpoint-url` is mandatory for a non-AWS
endpoint and `--recursive` is required to list a prefix.

**5. Cache and remote are in sync**

```console
$ dvc status -c
Cache and remote 's3' are in sync.
```

The message names `s3` because that is the resolved default remote — an indirect
confirmation of `core.remote` as well.

**6. The credentials are persisted where the criterion expects them**

```console
$ cat .dvc/config
[core]
    remote = s3
['remote "s3"']
    url = s3://dvc-storage
    endpointurl = http://localhost:8333
    access_key_id = weedadmin
    secret_access_key = weedadmin123
```

The prompt's `[!]` marker confirms `.dvc/config` is the modified, Git-tracked file — i.e. the
project scope, not `.dvc/config.local`.

## Gotchas

### `Unable to locate credentials` is a credential error, not a network error

DVC raised it before opening a connection: the S3 client refuses to sign a request with no
key. Debugging the endpoint or the bucket at that point is wasted effort — the first check is
the credential chain.

### The reported path is bucket + prefix, not a filesystem path

`(dvc-storage/files/md5)` in the error is the **remote** location DVC tried to list: bucket
`dvc-storage`, key prefix `files/md5`. It is not a directory on the local machine, and it is
not where the data would land — the workspace file is `data/raw/transactions.csv`.

### A clone carries the pointers, not the data

`git clone` brings `.dvc/config` and `data/raw/transactions.csv.dvc` (both committed) but
neither the dataset nor the cache (both git-ignored). That is the whole reason `dvc pull`
exists; a clone is *supposed* to be missing the data.

### `dvc remote modify` writes to the project config by default

Without a scope flag the keys land in the committed `.dvc/config`. `--local` targets the
git-ignored `.dvc/config.local`; `--global` / `--system` target the machine-wide files. Only
the first satisfies this lab's criterion.

### Credentials in `.dvc/config` are committed to Git

The lab deliberately asks for the keys in `.dvc/config`, and that is fine for a throwaway
exercise. In a real repository, keep the URL and endpoint in `.dvc/config` and move the
secrets to `.dvc/config.local` (`dvc remote modify --local`) or to `AWS_*` environment
variables — otherwise the credentials are published with the repository.

### `dvc pull` reads project *and* local config for credentials

`dvc pull` resolves credentials from the merged config (system → global → project → local).
Some other commands (notably `dvc list` on older versions) historically ignored the local
layer, so a credential that works for a pull may appear to be "ignored" elsewhere — a useful
distinction when a config looks right but a specific command still fails.

### `dvc status -c` is the cheapest end-to-end check

It resolves the remote, authenticates and lists objects without transferring data. If it
reports *in sync* while the workspace file is missing, the remote side is fine and the fix is
a `dvc pull` (fetch + checkout), not a config change.

### `dvc fetch` alone does not populate the workspace

`dvc fetch` only fills `.dvc/cache/`; the working file appears only after `dvc checkout` (or
a `dvc pull`, which does both). Seeing a populated cache but no `data/raw/transactions.csv`
is expected mid-way, not a failure.

### The `aws` CLI needs `--endpoint-url` here

Without it the CLI talks to `s3.amazonaws.com` and the credentials are rejected. The env-var
form (`AWS_ACCESS_KEY_ID=... AWS_SECRET_ACCESS_KEY=...`) is the quickest way to test an
endpoint independently of DVC and without touching `~/.aws/credentials`.

### `endpointurl`, not `endpoint_url`

The DVC config key has no underscore. It mirrors `boto3`'s `endpoint_url`, but writing the
underscored form in `.dvc/config` is silently treated as an unknown option — and the error
that follows (`Unable to locate credentials` against the AWS host) can look like a
credential problem rather than a typo.

### No `region` needed for SeaweedFS

Unlike AWS S3, SeaweedFS does not require a `region` option; the lab's pull succeeds without
one. Backends that do require it are configured with `dvc remote modify s3 region <region>`.

## Tutorial: diagnosing a failing `dvc pull` against an S3-compatible remote

A repeatable procedure for the *read* side of an S3-compatible DVC remote (Day 12 covers the
write side).

### Step 0 — Prerequisites

- DVC installed **with the S3 extra**: `pip install "dvc[s3]"`. Without the plugin DVC has no
  filesystem for `s3://` and fails with an unsupported-URL error before it looks at any
  option.
- A clone of a repository whose `.dvc` pointers and remote definition are committed.
- The dataset intentionally absent — that is what the pull is for.

### Step 1 — Read the error literally

| Message | Layer | Meaning |
| --- | --- | --- |
| `no remote specified` | config | no default remote resolved; fix with `dvc remote default` |
| `Unable to locate credentials` | auth | no credential found in the chain; fix the credentials |
| `NoSuchBucket` | storage | bucket does not exist |
| `Connection refused` / timeout | network | wrong endpoint URL or port |
| `SignatureDoesNotMatch` | auth | wrong secret, or clock skew |
| TLS error on an HTTP endpoint | transport | `use_ssl` still true |

### Step 2 — Confirm which remote resolves

```bash
dvc config --list | grep -E 'core.remote|endpointurl|url'
```

If `core.remote` is missing, no pull can proceed; if `endpointurl` is missing, DVC targets
AWS.

### Step 3 — Test the endpoint *outside* DVC

```bash
AWS_ACCESS_KEY_ID=weedadmin AWS_SECRET_ACCESS_KEY=weedadmin123 \
  aws --endpoint-url http://localhost:8333 s3 ls s3://dvc-storage/
```

A failure here is a backend or credential problem, not a DVC problem. A success here with a
DVC failure narrows the fault to DVC's own config.

### Step 4 — Fix the credentials at the right scope

| Approach | Command / source | Stored in | Use when |
| --- | --- | --- | --- |
| Local config | `dvc remote modify --local s3 access_key_id ...` | `.dvc/config.local` (git-ignored) | per-developer secrets — **default choice** |
| Environment | `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` / `AWS_SESSION_TOKEN` | process env | CI runners and containers |
| AWS profile | `dvc remote modify s3 profile myprofile` | `.dvc/config` (no secret) | shared `~/.aws/credentials` |
| Inline in project config | `dvc remote modify s3 secret_access_key ...` | `.dvc/config` (**committed**) | throwaway labs only |

For temporary credentials, also set `session_token`.

### Step 5 — Pull

```bash
dvc pull
```

Expect `N file(s) fetched and N file(s) added`. `dvc fetch` followed by `dvc checkout` is the
same thing split in two, useful when you want to inspect the cache before materialising the
workspace.

### Step 6 — Verify from the data side

```bash
cat data/raw/transactions.csv.dvc   # the expected md5
md5sum data/raw/transactions.csv    # must match
```

### Step 7 — Verify from the store side

```bash
aws --endpoint-url http://localhost:8333 s3 ls s3://dvc-storage/files/md5/ --recursive
```

The object name must recompose the pointer's hash. Finish with `dvc status -c` → *in sync*.

### Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `Unable to locate credentials` | credentials not visible to DVC | `dvc remote modify s3 …` (or `--local`), or export the `AWS_*` env vars |
| `no remote specified` | no default remote | `dvc remote default <name>` or `dvc pull -r <name>` |
| `Unsupported URL` / no filesystem for scheme | `dvc-s3` plugin missing | `pip install "dvc[s3]"` |
| `NoSuchBucket` | bucket does not exist | create it with `aws s3 mb` |
| Connection refused / timeout | wrong endpoint URL or port | fix `endpointurl`; confirm with `aws s3 ls` |
| `SignatureDoesNotMatch` | wrong secret, or clock skew | re-check the secret; sync the host clock |
| Pull "succeeds" but the file is absent | only `dvc fetch` ran | run `dvc checkout` or a full `dvc pull` |
| `dvc status -c` in sync but file missing | cache populated, workspace not | `dvc checkout` |

### Pre-flight checklist

1. `dvc remote list` shows the remote with the intended URL.
2. `dvc config --list | grep endpointurl` shows the intended host and port.
3. `dvc config --list | grep core` shows the intended default remote.
4. The credentials resolve (inline, env vars, or `~/.aws/credentials`).
5. No secret sits in the committed `.dvc/config` (in a real repository).
6. `aws s3 ls` reaches the bucket with the same credentials.

## Further reading

### Books

- **The Pragmatic Programmer for Machine Learning** — Marco Scutari & Mauro Malvestio
  (Chapman & Hall/CRC, 2023). **Open access** at <https://ppml.dev/>. Chapter 10,
  *"Tools for Developing Pipelines"*, covers data versioning and provenance, and the
  storage-layout reasoning behind separating pointers (Git) from payloads (a remote).
- **Building Machine Learning Pipelines** — Hannes Hapke & Catherine Nelson (O'Reilly,
  2020), <https://www.oreilly.com/library/view/building-machine-learning/9781492053187/>.
  Chapter 3, *"Data Ingestion"*, frames the "where does the data live" problem that a remote
  answers — and, by extension, why a clone must pull it back.
- **Pro Git** — Scott Chacon & Ben Straub (Apress, 2014), free at
  <https://git-scm.com/book/en/v2>. Not about S3, but chapter 2 explains the Git-side
  contract (what is committed, what is ignored) that makes a fresh clone data-less.
- *Note:* as with the other days, no book covers S3-compatible DVC remotes exclusively; the
  chapters above are the closest coverage inside broader texts.

### Official documentation

- `dvc pull` — <https://doc.dvc.org/command-reference/pull> — the fetch+checkout composition
  used throughout this write-up.
- `dvc fetch` — <https://doc.dvc.org/command-reference/fetch> — remote → cache only.
- `dvc checkout` — <https://doc.dvc.org/command-reference/checkout> — cache → workspace only.
- `dvc remote modify` — <https://doc.dvc.org/command-reference/remote/modify> — including the
  `--local` scope recommended for credentials.
- `dvc config` — <https://doc.dvc.org/command-reference/config> — the four config levels and
  `--list`.
- `dvc status` — <https://doc.dvc.org/command-reference/status> — `-c` / `--cloud` and the
  *in sync* message used in verification.
- Remote storage user guide —
  <https://doc.dvc.org/user-guide/data-management/remote-storage> — the canonical overview
  of supported storage types and their options.
- boto3 credentials —
  <https://boto3.amazonaws.com/v1/documentation/api/latest/guide/credentials.html> — the
  credential chain DVC inherits, which is why `Unable to locate credentials` is the exact
  symptom of an empty chain.
- s3fs — <https://s3fs.readthedocs.io/en/latest/> — the filesystem layer underneath DVC's
  S3 remote.
- SeaweedFS — <https://github.com/seaweedfs/seaweedfs> and its S3 API notes,
  <https://github.com/seaweedfs/seaweedfs/wiki/Amazon-S3-API> — the backend used here.
- `aws s3` CLI — <https://docs.aws.amazon.com/cli/latest/reference/s3/> — `ls --recursive`
  and `--endpoint-url` as used in the tutorial.

### Blog articles and discussions

- DVC issue #10064, *dvc pull: failed to load directory when first failed s3 connection* —
  <https://github.com/iterative/dvc/issues/10064> — shows the same
  `failed to connect to s3 (…/files/md5) - …` error shape reproduced here.
- DVC issue #5141, *s3 credentials in the repo level dvc config are not recognized* —
  <https://github.com/iterative/dvc/issues/5141> — why some commands historically ignored
  the local config layer while `push` / `pull` use it.
- DVC Discuss — <https://discuss.dvc.org/> — the forum where credential-chain questions for
  MinIO and similar backends are answered by the maintainers.
- DVC repository — <https://github.com/treeverse/dvc> (formerly `iterative/dvc`) — the
  `dvc remote` / config implementation and the S3 option schema live here.
- `dvc-s3` plugin — <https://github.com/iterative/dvc-s3> — the package that supplies the
  `s3://` filesystem.

### Video tutorials

- **DVCorg — "Sharing Data and Models with DVC (Hands-On Data Science Tutorial!)"** —
  <https://www.youtube.com/watch?v=EE7Gk84OZY8> — remotes and `dvc push` / `dvc pull` from
  the ground up.
- **DVCorg — "Reaching the next level of ML experiment tracking with the DVC 3.0 Stack"** —
  <https://www.youtube.com/watch?v=u-URI5Lvc-g> — the 3.x remote and cache model, i.e. the
  generation whose `files/md5/` layout this day verifies.

## Repository notes

- This directory documents the exercise; the project itself lives at
  `/root/code/fraud-detection/` and is not committed here.
- `task.txt` is the exercise brief and is kept local only (see `.gitignore`).
- The transcript above is the actual session, including the initial failure and the corrected
  configuration.
- Companion to Day 12: that day fixed the *write* side (`dvc push`), this day fixes the
  *read* side (`dvc pull`) against the same SeaweedFS bucket.
- Environment verified for this write-up: DVC `3.67.1`, where the S3 config schema defines
  `endpointurl`, the credential keys are `access_key_id` / `secret_access_key`, and the cache
  stores an uncompressed copy (`files/md5/...`).
