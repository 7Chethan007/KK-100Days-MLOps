# Day 12 — Fix a Broken DVC Remote and Push to SeaweedFS

A KodeKloud "100 MLOps" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive the fix
yourself*.

This is the direct sequel to Day 11 (`dvc add`) — that lab put a dataset
under DVC's management locally; this lab gets it into **remote**
storage (SeaweedFS, an S3-compatible object store), which is where
`dvc push`/`dvc pull` actually make a dataset shareable across machines.

---

## 1. Scenario

The `fraud-detection` project already has DVC initialized, with
`data/raw/transactions.csv` tracked locally and a remote named `s3`
configured — but broken. SeaweedFS (S3-compatible) is running locally
with an existing bucket.

| Component | Required value |
|---|---|
| S3 endpoint | `http://localhost:8333` |
| Bucket | `dvc-storage` |
| DVC remote name | `s3` |
| Default remote | `s3` |

Required end state: `dvc push` succeeds, and SeaweedFS contains at
least one object under `files/md5/...`.

---

## 2. Reasoning model — how to *derive* the fix, not memorize it

### 2.1 Three layers, three separate questions

```text
Dataset
   │
   ▼
DVC (tracks identity via content hash)
   │
   │ dvc push
   ▼
DVC Remote "s3"  (a NAME DVC refers to)
   │
   │ S3 protocol
   ▼
S3 Endpoint: http://localhost:8333  (WHERE the server is)
   │
   ▼
SeaweedFS → dvc-storage bucket  (WHICH container inside that server)
   │
   ▼
files/md5/82/...
```

Four genuinely independent configuration questions hide inside "fix the
DVC remote":
1. *Which remote name* does `dvc push` use by default?
2. *Which bucket*?
3. *Which S3 server* (endpoint)?
4. *Which credentials*?

Treating "the remote is broken" as one monolithic problem, instead of
checking each of these four independently, is exactly how you'd miss
that this lab actually had three separate wrong values, not one.

### 2.2 S3 is a protocol, not a specific vendor — why SeaweedFS fits in at all

DVC speaks "S3" the same way a browser speaks "HTTP" — it doesn't need
to know it's talking to SeaweedFS specifically, only that the server on
the other end implements the S3 API. This is why the DVC-facing URL uses
`s3://dvc-storage` (S3 semantics) while the actual server address is
`http://localhost:8333` (SeaweedFS's local endpoint) — **the scheme
(`s3://`) and the endpoint (`http://localhost:8333`) are two different
pieces of information**, not two ways of writing the same thing.

### 2.3 Endpoint vs. bucket — don't conflate "where" with "which container"

```text
endpointurl  →  WHICH SERVER to talk to        (http://localhost:8333)
url (bucket) →  WHICH CONTAINER on that server (s3://dvc-storage)
```

Getting one right and the other wrong still fails completely — DVC
needs both correct simultaneously to reach the right objects. This
lab's broken config had *both* wrong independently (`dvc-wrong-bucket`
and `:9999`), which is exactly why diagnosing "the remote" required
checking each field, not assuming a single typo.

### 2.4 `dvc status` healthy ≠ remote healthy — two different scopes of "health"

```bash
dvc status
```
```text
Data and pipelines are up to date.
```

`dvc status` compares your **working tree against local DVC metadata** —
it has nothing to say about remote connectivity, remote credentials, or
whether the remote even exists correctly. A clean `dvc status` here is
real information (local state is fine), but it's not evidence the
remote configuration is also fine — those are different scopes of
"health" entirely, checked by different commands.

### 2.5 Reproduce the actual failure before touching config

```bash
dvc push -v
```
```text
ERROR: failed to push data to the cloud - config file error:
no remote specified in /root/code/fraud-detection.
Setup default remote with
    dvc remote default <remote name>
```

Same "observe the real error before guessing a fix" discipline as every
broken-config lab in this series (Day 3's lockfile, Day 5's Makefile,
Day 8's pre-commit config). The error here is unusually precise — DVC
doesn't just say "push failed," it names the exact missing piece
(`no remote specified`) and tells you the exact command to fix it.

### 2.6 Why a *default* remote is a separate concept from a remote merely *existing*

```text
dvc remote list   →  s3   s3://dvc-wrong-bucket     (the remote EXISTS)
dvc push          →  "no remote specified"            (but isn't the DEFAULT)
```

A project can have several configured remotes (`s3-prod`, `minio-local`,
`azure`, etc.) — running bare `dvc push` with no `-r <name>` flag
requires DVC to know *which one to assume*. That assumption is exactly
what `[core] remote = s3` in `.dvc/config` encodes, and the broken
config was missing it entirely — the remote `s3` existed, but nothing
told DVC to use it by default.

### 2.7 Inspecting `.dvc/config` directly — compare field by field against the spec

```ini
['remote "s3"']
    url = s3://dvc-wrong-bucket        ← WRONG (required: dvc-storage)
    endpointurl = http://localhost:9999  ← WRONG (required: 8333)
    access_key_id = weedadmin           ← already correct
    secret_access_key = weedadmin123    ← already correct
```

Listing every field against the lab's stated requirements, one at a
time, rather than assuming "it's probably just the bucket" or "probably
just the endpoint," is what correctly surfaces that *two* fields were
wrong simultaneously, plus the missing default (§2.6) — three
independent problems, exactly like Day 3's broken `requirements.in` had
three independent root causes rather than one.

### 2.8 `dvc remote modify` — change exactly one field at a time

```bash
dvc remote modify s3 url s3://dvc-storage
dvc remote modify s3 endpointurl http://localhost:8333
dvc remote default s3
```

Each command targets exactly one named field on the named remote — no
need to hand-edit the TOML/INI-style `.dvc/config` directly (though you
could); DVC's own CLI is the safer way to make each correction
individually and verifiably.

### 2.9 Re-verify the config file itself before retrying the operation

```bash
cat .dvc/config
```
```ini
[core]
    remote = s3
['remote "s3"']
    url = s3://dvc-storage
    endpointurl = http://localhost:8333
    access_key_id = weedadmin
    secret_access_key = weedadmin123
```

Confirming the actual file contents — not just trusting that three
`dvc remote modify`/`dvc remote default` commands "probably worked" —
before spending time on `dvc push` again. Same "verify the fix landed"
discipline as every lab in this series.

### 2.10 Reading `dvc push -v`'s own trace as proof of correct wiring

```text
Preparing to transfer data from
'/root/code/fraud-detection/.dvc/cache/files/md5'
to
's3://dvc-storage/files/md5'
...
1 file pushed
```

This single debug line proves the entire local-to-remote path resolved
correctly — the exact local cache path, and the exact remote bucket
path, both shown explicitly. `1 file pushed` is the headline result, but
the path trace above it is what actually confirms *which* object went
*where*.

### 2.11 Why remote objects land under `files/md5/<prefix>/<rest>`

```text
Local:  .dvc/cache/files/md5/82/564f01ab73aa3860e11327cc14be2c
Remote: s3://dvc-storage/files/md5/82/564f01ab73aa3860e11327cc14be2c
```

DVC's content-addressable cache splits an object's MD5 hash into a
2-character prefix directory plus the remaining hash as the filename —
avoiding, at scale, millions of files sitting flat in one directory
(a classic filesystem/object-store scaling problem). The remote mirrors
this exact layout, so `files/md5/...` showing up in SeaweedFS after a
push is expected structure, not an arbitrary or surprising path.

### 2.12 Verification should check the actual backend, not just DVC's own "it worked" message

```bash
curl -s http://localhost:8888/buckets/dvc-storage/files/md5/ | grep -oE '82[^"<]*' | head -20
curl -s -o /dev/null -w 'HTTP=%{http_code}\n' \
  http://localhost:8888/buckets/dvc-storage/files/md5/82/564f01ab73aa3860e11327cc14be2c
```
```text
HTTP=200
```

`dvc push` reporting `1 file pushed` is DVC's own claim about what it
did — querying SeaweedFS's Filer UI directly (a completely independent
system) and getting `HTTP 200` back for the *exact* object path is
external, independent proof the object genuinely exists in the backend.
This is the same "verify the real resource state, not just the tool's
own success message" discipline running through this entire series,
here checked against a system DVC itself doesn't control.

### 2.13 The compressed reasoning chain

```text
Requirement (s3 remote → s3://dvc-storage, endpoint :8333, default remote, pushed, verifiable in SeaweedFS)
   → dvc status                               → local state healthy (doesn't prove remote health — §2.4)
   → dvc push -v                                → FAILS: "no remote specified" (precise, actionable error)
   → cat .dvc/config                             → compare EVERY field against the spec
        → bucket wrong (dvc-wrong-bucket)
        → endpoint wrong (:9999)
        → credentials already correct
        → no [core] remote = s3 at all
   → dvc remote modify s3 url s3://dvc-storage
   → dvc remote modify s3 endpointurl http://localhost:8333
   → dvc remote default s3
   → cat .dvc/config again                        → confirm all 4 fields now correct
   → dvc push -v                                   → trace shows LOCAL path → REMOTE path correctly; "1 file pushed"
   → Independently verify via SeaweedFS Filer:     → object listing shows the hash prefix; direct GET returns HTTP 200
   → dvc status again                              → still "up to date" (local AND remote now in sync)
```

---

## 3. Concepts (reference)

### 3.1 DVC remote vs. DVC cache (recap, extended from Day 11)
The **cache** (`.dvc/cache/`) is local storage for DVC-tracked objects.
The **remote** is shared storage (here, SeaweedFS over S3) that
`dvc push`/`dvc pull` synchronize the cache against — Day 11 only ever
populated the cache; this lab is the first time data leaves the local
machine.

### 3.2 Content-addressable storage
Objects are identified by a hash of their own content (MD5 here), not by
an arbitrary name or path. Identical content always hashes identically,
letting DVC (and SeaweedFS) recognize "I already have this exact data"
without a byte-by-byte comparison.

### 3.3 S3 API vs. S3-the-product
"S3" names both Amazon's specific service and the de facto API many
other object stores (SeaweedFS, MinIO, Ceph, etc.) implement compatibly.
A tool coded against "the S3 API" can talk to any S3-compatible backend
by pointing its endpoint at that backend instead of AWS's.

### 3.4 `.dvc/config` structure
```ini
[core]
    remote = <default-remote-name>
['remote "<name>"']
    url = s3://<bucket>
    endpointurl = <server-address>
    access_key_id = ...
    secret_access_key = ...
```
`[core] remote` picks the default; each `['remote "<name>"']` block
fully describes one named remote's own connection details.

### 3.5 `dvc remote modify` vs. hand-editing `.dvc/config`
Both ultimately change the same file; the CLI form is safer for
one-field-at-a-time corrections since it validates the specific field
name/remote name rather than risking a manual syntax error in the raw
config file.

### 3.6 Why verification should span two independent systems
Confirming success through DVC's own output (`1 file pushed`,
`dvc status`) *and* through the backend object store itself (SeaweedFS
Filer HTTP response) is stronger than trusting either alone — the same
principle as checking both sides of an AWS resource relationship
(Days 10, 11, 12 in the Cloud-AWS track) applied here to a tool-and-
backend pair instead of two AWS resources.

---

## 4. Runbook

### 4.1 Confirm DVC and local state
```bash
cd /root/code/fraud-detection
dvc --version
```
```text
3.67.1
```
```bash
dvc status
```
```text
Data and pipelines are up to date.
```

### 4.2 Inspect the current (broken) remote config
```bash
cat .dvc/config
```
```ini
['remote "s3"']
    url = s3://dvc-wrong-bucket
    endpointurl = http://localhost:9999
    access_key_id = weedadmin
    secret_access_key = weedadmin123
```
```bash
dvc remote list
```
```text
s3      s3://dvc-wrong-bucket
```

### 4.3 Reproduce the failure
```bash
dvc push -v
```
```text
ERROR: failed to push data to the cloud - config file error:
no remote specified in /root/code/fraud-detection.
Setup default remote with
    dvc remote default <remote name>
```

### 4.4 Fix the bucket
```bash
dvc remote modify s3 url s3://dvc-storage
```

### 4.5 Fix the endpoint
```bash
dvc remote modify s3 endpointurl http://localhost:8333
```

### 4.6 Set the default remote
```bash
dvc remote default s3
```

### 4.7 Verify the corrected configuration
```bash
cat .dvc/config
```
```ini
[core]
    remote = s3
['remote "s3"']
    url = s3://dvc-storage
    endpointurl = http://localhost:8333
    access_key_id = weedadmin
    secret_access_key = weedadmin123
```
```bash
dvc remote list
```
```text
s3      s3://dvc-storage
(default)
```

### 4.8 Push
```bash
dvc push -v
```
```text
Preparing to transfer data from
'/root/code/fraud-detection/.dvc/cache/files/md5'
to
's3://dvc-storage/files/md5'
...
1 file pushed
```

### 4.9 Confirm local DVC sees everything as in sync
```bash
dvc status
```
```text
Data and pipelines are up to date.
```

### 4.10 Identify the exact object to verify remotely
```bash
find .dvc/cache/files/md5 -type f -print
```
```text
.dvc/cache/files/md5/82/564f01ab73aa3860e11327cc14be2c
```
```bash
find .dvc/cache/files/md5 -type f | wc -l
```
```text
1
```

### 4.11 Verify via SeaweedFS directly — listing
```bash
curl -s http://localhost:8888/buckets/dvc-storage/files/md5/ \
  | grep -oE '82[^"<]*' \
  | head -20
```
```text
82/
82
```

### 4.12 Verify via SeaweedFS directly — the exact object
```bash
curl -s -o /dev/null -w 'HTTP=%{http_code}\n' \
  http://localhost:8888/buckets/dvc-storage/files/md5/82/564f01ab73aa3860e11327cc14be2c
```
```text
HTTP=200
```

### 4.13 Submit
With the config matching every required field, `dvc push` reporting
`1 file pushed`, and SeaweedFS independently confirming the object via
`HTTP=200`, click **Check** in the KodeKloud lab UI.

---

## 5. Final state

| Requirement | Result |
|---|---|
| Remote `s3` → bucket `dvc-storage` | ✅ (was `dvc-wrong-bucket`) |
| Endpoint `http://localhost:8333` | ✅ (was `:9999`) |
| Credentials | ✅ (already correct) |
| Default remote = `s3` | ✅ (was missing entirely) |
| `dvc push` succeeds | ✅ `1 file pushed` |
| Object present in SeaweedFS under `files/md5/...` | ✅ `HTTP=200` |

```text
Before:                              After:
DVC remote                            DVC remote
 ├── bucket ❌ dvc-wrong-bucket        ├── bucket ✅ dvc-storage
 ├── endpoint ❌ localhost:9999        ├── endpoint ✅ localhost:8333
 ├── credentials ✅                    ├── credentials ✅
 └── default remote ❌ (none)          └── default remote ✅ s3
                                              │
                                         dvc push
                                              │
                                              ▼
                                   SeaweedFS: dvc-storage/files/md5/82/...
                                              │
                                              ▼
                                         HTTP 200
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `dvc status` says everything is fine, but `dvc push` fails | `dvc status` only checks LOCAL working-tree/metadata sync, never remote connectivity | Always test `dvc push`/`dvc pull` directly to validate the remote, don't infer remote health from `dvc status` |
| `ERROR: no remote specified` | A remote exists but isn't set as the default, and no `-r <name>` was passed | `dvc remote default <name>`, or explicitly pass `-r <name>` to `dvc push` |
| `dvc push` fails to connect at all | Wrong `endpointurl` — pointed at a server/port nothing is listening on | `dvc remote modify <name> endpointurl <correct-url>`; re-verify with `cat .dvc/config` |
| `dvc push` connects but objects land in the wrong place / 404s on verification | Wrong bucket in the `url` field | `dvc remote modify <name> url s3://<correct-bucket>` |
| Only fixed one of two wrong config fields, then declared success | Assumed a single typo instead of checking every field against the spec | List every remote field and compare each one individually (§2.7) before retrying |
| `dvc push` reports success but you're not sure the object is really there | Trusted DVC's own report without checking the backend independently | Query the object store directly (SeaweedFS Filer, S3 `ls`, etc.) for the exact expected path |
| Hand-edited `.dvc/config` and introduced a syntax error | Manual edits to an INI/TOML-like file risk typos `dvc remote modify` wouldn't make | Prefer `dvc remote modify`/`dvc remote default` over direct file edits when correcting individual fields |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.13) out loud
      from "dvc push fails with no remote specified" to "verified via
      an independent HTTP 200 against SeaweedFS."
- [ ] Explain, in one sentence, why `dvc status` being clean didn't mean
      the remote was correctly configured.
- [ ] Explain the difference between a remote *existing* in
      `dvc remote list` and being the *default* remote — why does
      `dvc push` care about the difference?
- [ ] Explain the difference between `endpointurl` and `url` (bucket) in
      a DVC remote config — what does each one actually control?
- [ ] Explain why DVC splits a content hash into a 2-character prefix
      directory plus the remaining hash as the filename.
- [ ] Explain why checking SeaweedFS's Filer directly is stronger
      verification than trusting `dvc push`'s own "1 file pushed"
      message.
- [ ] Break the config again (revert bucket, endpoint, and default
      remote all at once) and fix all three from memory, verifying each
      field individually before retrying `dvc push`.

---

## 8. Document template for future lab days

Use this skeleton for `day-N/README.md`:

```markdown
# Day N — <task title>

## 1. Scenario
<what was asked, what was broken/required, constraints>

## 2. Reasoning model — how to derive the fix/commands
<walk the requirement down step by step, in the order you'd actually
discover it: check local state → reproduce the actual failure → inspect
EVERY relevant config field against the spec individually → fix each →
re-verify the config itself → retry the operation → verify against the
independent backend, not just the tool's own success message. End with
a compressed arrow-chain version.>

## 3. Concepts (reference)
<one subsection per concept the task exercises — explain WHY, not just WHAT>

## 4. Runbook
<copy-pasteable commands in the order run, with real intermediate
output/results inline where they mattered to the diagnosis>

## 5. Final state
<table + diagram of what the configuration looks like after completion>

## 6. Troubleshooting
<symptom / cause / fix table>

## 7. Do it yourself (checklist)
<"walk the reasoning chain out loud" + comprehension questions + "break it
again and fix it from memory" prompt>
```
