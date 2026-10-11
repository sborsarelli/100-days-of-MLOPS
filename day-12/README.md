# Day 12 — Fixing a DVC Push to an S3-Compatible Remote (SeaweedFS)

## Overview

Part of the *100 Days of MLOps* challenge. The xFusionCorp Industries ML team keeps
[DVC](https://dvc.org/)-tracked data in a shared **SeaweedFS** S3-compatible object store
instead of in Git. The `fraud-detection` project already declares a remote called `s3`,
but `dvc push` refuses to run — it never even opens a connection.

The write-up documents the failure, the three independent defects behind it, the exact
commands used, the verification of both acceptance criteria, a reusable tutorial for
wiring *any* S3-compatible bucket to DVC, and pointers for deeper study.

## Task

The environment:

| Fact | Value |
| --- | --- |
| Project | `/root/code/fraud-detection/` (DVC initialised, `data/raw/transactions.csv` tracked) |
| Object store | SeaweedFS, S3 endpoint `http://localhost:8333` |
| Filer UI | forwarded port `8888`, buckets under `/buckets/` |
| Credentials | `weedadmin` / `weedadmin123` (already in `.dvc/config`) |
| Bucket | `dvc-storage` (already created) |

Acceptance criteria:

- The remote called `s3` points at the `dvc-storage` bucket using `s3://`, uses the correct
  SeaweedFS S3 endpoint URL, and is marked as the default remote.
- After the push, the `dvc-storage` bucket in the SeaweedFS Filer UI contains at least one
  object under the `files/md5/...` prefix.

## Background: DVC remotes and S3-compatible object stores

### What a DVC remote is

DVC keeps file content in a local **cache** and commits only a small pointer file
(`<path>.dvc`) to Git — the mechanism covered on Day 11. A **remote** is the second half of
that story: a storage location where `dvc push` uploads the cache and `dvc pull` downloads
it from. The remote is described entirely by a URL plus a handful of options; DVC stores
nothing about it outside its own config files.

### Where remotes are declared: `.dvc/config`

Remote definitions live in `.dvc/config`, an **INI-style file** (DVC uses `ConfigObj`, not
TOML). A section header such as `['remote "s3"']` declares the remote named `s3`:

```ini
['remote "s3"']
    url = s3://dvc-storage
    endpointurl = http://localhost:8333
    access_key_id = weedadmin
    secret_access_key = weedadmin123
```

DVC merges four configuration levels, each shadowing the previous one:

| Level | File | Committed? |
| --- | --- | --- |
| system | `/etc/dvc/config` | no |
| global | `~/.config/dvc/config` | no |
| project | `.dvc/config` | **yes** — this is the file the lab ships |
| local | `.dvc/config.local` | no (git-ignored by `.dvc/.gitignore`) |

That layering is why the same remote can be shared with credentials and then overridden
per-machine: the URL goes in `.dvc/config`, the secrets go in `.dvc/config.local`.

### The S3 keys that matter

| Key | Purpose |
| --- | --- |
| `url` | `s3://<bucket>/<optional/prefix>` — the bucket and the key prefix |
| `endpointurl` | host of the S3 API when it is **not** AWS |
| `access_key_id` / `secret_access_key` | credentials (or omit them and use the environment) |
| `session_token` | for temporary STS credentials |
| `region` | required by some backends, irrelevant to others |
| `use_ssl` | `true` by default; see the gotcha on plain-HTTP endpoints |
| `profile` / `credentialpath` | read credentials from `~/.aws/*` instead of inline values |

### Why `endpointurl` exists

The S3 REST API is a **de-facto standard**, not an AWS-only protocol. SeaweedFS, MinIO,
Ceph RGW, LocalStack, Garage and many others implement it. When no `endpointurl` is set,
DVC's S3 client targets `s3.amazonaws.com`; the option rewrites the host it talks to while
keeping the S3 wire protocol intact. This is the single most important knob for
self-hosted object storage.

Note the spelling: the DVC config key is **`endpointurl`** — one word, no underscore. It
mirrors the `endpoint_url` parameter of the underlying `boto3`/`s3fs` client, but the DVC
config key drops the underscore. Writing `endpoint_url` is silently treated as an unknown
option.

### How DVC resolves "which remote"

The resolution order is short, and the lab's error message names it exactly:

1. the `-r` / `--remote` flag passed to the command (e.g. `dvc push -r s3`);
2. the `core.remote` config value — the **default remote**;
3. otherwise: fail.

`core.remote` is what `dvc remote default s3` writes. Equivalently, `dvc config
core.remote s3` sets the same key by hand.

### The remote layout: `files/md5/...`

DVC 3.x hashes content with **md5** — visible in the pointer file as `hash: md5`. Objects
are stored under a two-level sharding scheme, prefixed by the hash algorithm:

```
files/md5/<first 2 hex chars>/<remaining 30 hex chars>
```

So the cache path is derived from the hash, never from the original filename: a
`transactions.csv` with hash `82564f01ab73aa3860e11327cc14be2c` lands at
`files/md5/82/564f01ab73aa3860e11327cc14be2c`. The `files/<algo>` prefix was introduced in
DVC 2.0; DVC 1.x used the bare `<2>/<30>` layout. Directory outputs additionally produce a
`.dir` JSON object in the same tree.

## Initial state

The remote exists but points nowhere useful, and no default is set:

```console
$ cd /root/code/fraud-detection/
$ dvc push
Collecting                           |0.00 [00:00,    ?entry/s]
Pushing
ERROR: failed to push data to the cloud - config file error: no remote specified in /root/code/fraud-detection. Setup default remote with
    dvc remote default <remote name>
or use:
    dvc <command> -r <remote name>

$ cat .dvc/config
['remote "s3"']
    url = s3://dvc-wrong-bucket
    endpointurl = http://localhost:9999
    access_key_id = weedadmin
    secret_access_key = weedadmin123
```

| Requirement | Initial state |
| --- | --- |
| URL is `s3://dvc-storage` | violated — `s3://dvc-wrong-bucket` |
| Endpoint is `http://localhost:8333` | violated — port `9999` |
| A default remote is set | missing |
| Credentials are correct | satisfied |

Three independent defects. Fixing only one still fails, and the first one to be reported is
the *missing default* — the push aborts before it ever tries to reach the endpoint.

## Solution

### 1. Correct the bucket

```bash
dvc remote modify s3 url s3://dvc-storage
```

### 2. Correct the endpoint

```bash
dvc remote modify s3 endpointurl http://localhost:8333
```

### 3. Mark the remote as default

```bash
dvc remote default s3
```

### 4. Confirm the resulting configuration

```console
$ dvc config --list
remote.s3.url=s3://dvc-storage
remote.s3.endpointurl=http://localhost:8333
remote.s3.access_key_id=weedadmin
remote.s3.secret_access_key=weedadmin123
core.remote=s3
```

The last line is the one that was missing: `core.remote=s3`. The equivalent edit by hand
would be to append a `['core']` section with `remote = s3` to `.dvc/config`.

## Verification

Run the following from `/root/code/fraud-detection`.

**1. The push succeeds**

```console
$ dvc push
Collecting                           |1.00 [00:00,  625entry/s]
Pushing
1 file pushed
```

One object is expected: the dataset is a single file, so DVC uploads one content object. A
`.dir` object would only appear for a directory output.

**2. The default remote is persisted**

```console
$ dvc config --list | grep core
core.remote=s3
```

**3. The object is in the bucket under `files/md5/`**

```console
$ AWS_ACCESS_KEY_ID=weedadmin AWS_SECRET_ACCESS_KEY=weedadmin123 \
    aws --endpoint-url http://localhost:8333 s3 ls s3://dvc-storage/files/md5/ --recursive
2026-10-11 13:26:12        218 files/md5/82/564f01ab73aa3860e11327cc14be2c
```

The same path is visible in the Filer UI under `/buckets/dvc-storage/files/md5/`. Note that
`--endpoint-url` is mandatory for a non-AWS endpoint and `--recursive` is required to list
a prefix.

**4. The object is the tracked dataset, not a leftover**

The strongest check is recomposing the hash recorded in the pointer file:

```console
$ cat data/raw/transactions.csv.dvc
outs:
- md5: 82564f01ab73aa3860e11327cc14be2c
  size: 218
  hash: md5
  path: transactions.csv
```

`82` + `564f01ab73aa3860e11327cc14be2c` is exactly the object name, and the recorded `size`
(218) matches the object's byte size (218) — see the gotcha on cache compression below.

**5. Cache and remote are in sync**

```console
$ dvc status -c
Cache and remote 's3' are in sync.
```

The message names `s3` because that is the resolved default remote — an indirect
confirmation of `core.remote` as well.

## Gotchas

### `no remote specified` is a config error, not a network error

The wording is literal: DVC did not resolve a remote, so it never contacted SeaweedFS.
Debugging the endpoint or the credentials at that point is wasted effort — the first fix is
`dvc remote default <name>`.

### `endpointurl`, not `endpoint_url`

The config key has no underscore. An unknown key in `.dvc/config` is not rejected loudly by
older versions, so a typo can look like "the endpoint is ignored".

### Three defects, three fixes

Wrong bucket, wrong port and a missing default are independent. The lab's error only
surfaces the last one first, which is a useful reminder that the reported symptom is not
always the root cause.

### Do not use `--local` for the default remote

`dvc remote default s3` writes to the committed `.dvc/config`. Adding `--local` would put
`core.remote` in `.dvc/config.local` instead — git-ignored, machine-specific, and invisible
to anyone else. Use `--local` for **secrets**, not for the shared topology.

### Credentials in `.dvc/config` are committed to Git

The lab deliberately ships `access_key_id` / `secret_access_key` inline, and that is fine
for a throwaway exercise. In a real repository, keep the URL and endpoint in
`.dvc/config` and move the credentials to `.dvc/config.local` (`dvc remote modify --local`)
or to environment variables.

### `dvc push` is idempotent

Running it again after a successful push prints `Everything is up to date` and uploads
nothing. That is success, not a failure.

### The bucket must exist before the push

DVC/s3fs creates key prefixes on demand but is not a reliable bucket-provisioning tool. The
bucket (`dvc-storage`) was created out-of-band and is visible in the Filer UI. Against AWS,
`aws s3 mb s3://bucket` plays the same role.

### `files/md5/...` will not contain your filename

The cache is content-addressed. Looking for `transactions.csv` in the bucket is the most
common false negative when verifying a push.

### No compression: 218 bytes really is 218 bytes

DVC 3.67.1 stores a byte-for-byte copy of the file in the cache — there is no gzip layer.
The `size` field in the pointer file is the real on-disk size, so it must equal the object
size in the bucket. A mismatch there would mean the object is not the tracked content.

### `use_ssl` and plain-HTTP endpoints

`use_ssl` defaults to `true`, and DVC honours the `http://` scheme in `endpointurl`. In this
lab the push worked without touching it. If a plain-HTTP endpoint produces TLS errors
(`SSL: WRONG_VERSION_NUMBER`), the escape hatch is:

```bash
dvc remote modify s3 use_ssl false
```

### The `aws` CLI needs `--endpoint-url` here

Without it, the CLI talks to `s3.amazonaws.com` and the credentials are rejected. The
env-var form (`AWS_ACCESS_KEY_ID=... AWS_SECRET_ACCESS_KEY=...`) is the quickest way to
test an endpoint without touching `~/.aws/credentials`.

### `.dvc/config` is INI, not YAML or TOML

The pointer file `<path>.dvc` *is* YAML; `.dvc/config` is not. Editing it by hand means
matching `['remote "name"']` headers and 4-space indentation — a stray quote makes DVC skip
the section silently. The `dvc remote` commands exist precisely to avoid that class of
error.

## Tutorial: connecting an S3-compatible bucket to DVC

A repeatable procedure for wiring any S3-compatible object store to a DVC project.

### Step 0 — Prerequisites

- DVC installed **with the S3 extra**: `pip install "dvc[s3]"`. Without the plugin, DVC has
  no filesystem for the `s3://` scheme and fails with an unsupported-URL error before it
  looks at any option.
- A Git repository with DVC initialised (`dvc init`).
- At least one path tracked by DVC (`dvc add data/...`), otherwise there is nothing to push.

### Step 1 — Collect the five facts

| Fact | Example (SeaweedFS) | Where it comes from |
| --- | --- | --- |
| Bucket name | `dvc-storage` | backend console / Filer UI |
| Optional key prefix | *(none)* | your convention, e.g. `dvc-store/` |
| Endpoint URL | `http://localhost:8333` | backend config (SeaweedFS: `-s3.port`) |
| Access key | `weedadmin` | backend config |
| Secret key | `weedadmin123` | backend config |

### Step 2 — Create the bucket (outside DVC)

Do it with the backend's own tooling or an S3 client:

```bash
AWS_ACCESS_KEY_ID=weedadmin AWS_SECRET_ACCESS_KEY=weedadmin123 \
  aws --endpoint-url http://localhost:8333 s3 mb s3://dvc-storage
```

Verify it is reachable *before* involving DVC:

```bash
AWS_ACCESS_KEY_ID=weedadmin AWS_SECRET_ACCESS_KEY=weedadmin123 \
  aws --endpoint-url http://localhost:8333 s3 ls s3://dvc-storage/
```

A failure here is a backend or credential problem, not a DVC problem.

### Step 3 — Declare the remote

```bash
dvc remote add -d s3 s3://dvc-storage
```

`-d` marks it as the default remote in the same step. Add a prefix if the bucket is shared:
`s3://dvc-storage/fraud-detection`.

### Step 4 — Point at the non-AWS endpoint

```bash
dvc remote modify s3 endpointurl http://localhost:8333
```

Skip this only when the target really is AWS.

### Step 5 — Provide credentials

Four options, in decreasing order of how much they leak:

| Approach | Command / source | Stored in | Use when |
| --- | --- | --- | --- |
| Local config | `dvc remote modify --local s3 access_key_id ...` | `.dvc/config.local` (git-ignored) | per-developer secrets — **default choice** |
| Environment | `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` / `AWS_SESSION_TOKEN` | process env | CI runners and containers |
| AWS profile | `dvc remote modify s3 profile myprofile` | `.dvc/config` (no secret) | shared `~/.aws/credentials` |
| Inline in project config | `dvc remote modify s3 secret_access_key ...` | `.dvc/config` (**committed**) | throwaway labs only |

For temporary credentials, also set `session_token`. For backends that require a region,
add `dvc remote modify s3 region <region>`.

### Step 6 — Make it the default (if Step 3 did not)

```bash
dvc remote default s3
```

### Step 7 — Smoke test, then push

```bash
dvc status -c     # compares cache against the remote without transferring data
dvc push
```

`dvc status -c` is the cheapest end-to-end check: it resolves the remote, authenticates and
lists objects. If it reports *in sync* on an empty bucket, the pointer and the remote are
consistent but nothing has been uploaded yet — run `dvc push` to be sure.

### Step 8 — Verify from the store side

```bash
aws --endpoint-url http://localhost:8333 s3 ls s3://dvc-storage/files/md5/ --recursive
```

You should see objects under `files/md5/<2>/<30>`. Cross-check at least one against the
`md5` in the corresponding `.dvc` file.

### Recipes per backend

| Backend | Default S3 endpoint | Remote setup |
| --- | --- | --- |
| AWS S3 | *(none — omit `endpointurl`)* | `dvc remote add -d s3 s3://bucket/prefix` + `dvc remote modify s3 region eu-west-1` |
| SeaweedFS | `http://localhost:8333` | `dvc remote modify s3 endpointurl http://localhost:8333` |
| MinIO | `http://localhost:9000` | `dvc remote modify minio endpointurl http://localhost:9000` |
| Ceph RGW | `http://rgw.internal:7480` | `dvc remote modify ceph endpointurl http://rgw.internal:7480` |
| LocalStack | `http://localhost:4566` | `dvc remote modify local endpointurl http://localhost:4566` |

### Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `no remote specified` | no default remote | `dvc remote default <name>` or `dvc push -r <name>` |
| `Unsupported URL` / no filesystem for scheme | `dvc-s3` plugin missing | `pip install "dvc[s3]"` |
| `Unable to locate credentials` | credentials not visible to DVC | `dvc remote modify --local …` or export the AWS env vars |
| `NoSuchBucket` | bucket does not exist | create it (Step 2) |
| Connection refused / timeout | wrong endpoint URL or port | fix `endpointurl`; confirm with `aws s3 ls` |
| `SignatureDoesNotMatch` | wrong secret, or clock skew | re-check the secret; sync the host clock |
| TLS error on an HTTP endpoint | `use_ssl` still true | `dvc remote modify <name> use_ssl false` |
| Push "succeeds" but the bucket looks empty | wrong bucket, wrong prefix, or `files/md5/` not inspected | `dvc status -c`, then list `files/md5/` recursively |

### Pre-flight checklist

1. Bucket exists and is reachable with `aws s3 ls` using the same credentials.
2. `dvc remote list` shows the remote with the intended URL.
3. `dvc config --list | grep endpointurl` shows the intended host and port.
4. `dvc config --list | grep core` shows the intended default remote.
5. No secret sits in the committed `.dvc/config`.
6. `dvc status -c` runs without an error.

## Further reading

### Books

- **The Pragmatic Programmer for Machine Learning** — Marco Scutari & Mauro Malvestio
  (Chapman & Hall/CRC, 2023). **Open access** at <https://ppml.dev/>. Chapter 10,
  *"Tools for Developing Pipelines"*, covers data versioning and provenance, and the
  storage-layout reasoning behind separating pointers (Git) from payloads (a remote).
- **Building Machine Learning Pipelines** — Hannes Hapke & Catherine Nelson (O'Reilly,
  2020), <https://www.oreilly.com/library/view/building-machine-learning/9781492053187/>.
  Chapter 3, *"Data Ingestion"*, frames the "where does the data live" problem that a remote
  answers.
- **Pro Git** — Scott Chacon & Ben Straub (Apress, 2014), free at
  <https://git-scm.com/book/en/v2>. Not about S3, but chapter 2 explains the Git-side
  contract (what is committed, what is ignored) that a DVC remote depends on.
- *Note:* as with the other days, no book covers S3-compatible DVC remotes exclusively; the
  chapters above are the closest coverage inside broader texts.

### Official documentation

- `dvc remote add` — <https://doc.dvc.org/command-reference/remote/add> — the `-d` flag and
  URL semantics.
- `dvc remote modify` — <https://doc.dvc.org/command-reference/remote/modify> — including
  the `--local` scope used for credentials.
- `dvc remote default` — <https://doc.dvc.org/command-reference/remote/default> — the
  `core.remote` key set in this exercise.
- Remote storage user guide —
  <https://doc.dvc.org/user-guide/data-management/remote-storage> — the canonical overview
  of supported storage types and their options.
- `dvc push` — <https://doc.dvc.org/command-reference/push> — the `-r` flag and the
  idempotent behaviour.
- `dvc status` — <https://doc.dvc.org/command-reference/status> — `-c` / `--cloud` and the
  *in sync* message used in verification.
- SeaweedFS — <https://github.com/seaweedfs/seaweedfs> and its S3 API notes,
  <https://github.com/seaweedfs/seaweedfs/wiki/Amazon-S3-API> — the backend used here.
- MinIO — <https://min.io/docs/minio/linux/index.html> — the most common self-hosted
  alternative; its S3 API docs are a good cross-reference for `endpointurl` behaviour.
- Ceph RGW — <https://docs.ceph.com/en/latest/radosgw/s3/> — the S3 API surface of Ceph's
  object gateway.
- LocalStack S3 — <https://docs.localstack.cloud/user-guide/aws/s3/> — the fastest way to
  rehearse this tutorial without a real bucket.
- boto3 credentials —
  <https://boto3.amazonaws.com/v1/documentation/api/latest/guide/credentials.html> — the
  credential chain DVC inherits, which is why the `AWS_*` environment variables work.
- s3fs — <https://s3fs.readthedocs.io/en/latest/> — the filesystem layer underneath DVC's
  S3 remote, and the source of the `use_ssl` / `endpoint_url` options.
- `aws s3` CLI — <https://docs.aws.amazon.com/cli/latest/reference/s3/> — `ls --recursive`
  and `mb` as used in the tutorial.

### Blog articles and discussions

- DVC Discuss — <https://discuss.dvc.org/> — the forum where `endpointurl` questions for
  MinIO and similar backends are answered by the maintainers.
- DVC repository — <https://github.com/treeverse/dvc> (formerly `iterative/dvc`) — the
  `dvc remote` implementation and the S3 config schema live here.
- `dvc-s3` plugin — <https://github.com/iterative/dvc-s3> — the package that supplies the
  `s3://` filesystem; the place to look when a config key appears to be ignored.

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
- The transcript above is the actual session, including the initial failure and the
  corrected configuration.
- Environment verified for this write-up: DVC `3.67.1`, where the S3 config schema defines
  `endpointurl` and defaults `use_ssl` to `true`.
