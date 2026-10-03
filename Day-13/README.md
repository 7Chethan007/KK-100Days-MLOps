# Day 13 — Pull DVC-Tracked Data from Remote

A KodeKloud "100 MLOps" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive the fix
yourself*.

This is the direct sequel to Day 12 (fixing a broken `dvc push`) — same
remote, same SeaweedFS backend, but today's failure is on the *pull*
side of a freshly-cloned project, and the root cause is a different
layer of the same configuration entirely.

---

## 1. Scenario

A cloned `fraud-detection` project has the DVC pointer
`data/raw/transactions.csv.dvc`, but the actual dataset file is
missing locally. The data lives in the same SeaweedFS S3-compatible
remote from Day 12. Task:

1. Diagnose why `dvc pull` fails.
2. Fix the DVC remote's authentication configuration.
3. Pull the dataset.
4. Verify the downloaded file's content hash matches what the `.dvc`
   pointer expects.

| Component | Value |
|---|---|
| Project | `/root/code/fraud-detection/` |
| DVC remote | `s3://dvc-storage` |
| S3 endpoint | `http://localhost:8333` |
| Access key | `weedadmin` |
| Secret key | `weedadmin123` |
| Dataset | `transactions.csv` |

---

## 2. Reasoning model — how to *derive* the fix, not memorize it

### 2.1 A fresh clone has metadata but not data — this is expected, not broken

```text
git clone  →  you get: source code, .dvc POINTER files
           →  you do NOT get: the actual DVC-tracked data (never committed to Git)
```

This is the entire point of the Git/DVC split established in Days 10–11:
Git stores the lightweight pointer; the actual bytes live in the DVC
remote. A freshly-cloned repo missing `transactions.csv` on disk isn't a
defect — it's the normal starting state, and `dvc pull` is the step that
was always going to be needed to materialize the real file.

### 2.2 `dvc status` tells you *what's* missing, precisely

```bash
dvc status
```
```text
data/raw/transactions.csv.dvc:
    changed outs:
        not in cache: data/raw/transactions.csv
```

`not in cache` is specific: DVC knows exactly which file it expects
(from the `.dvc` pointer) and confirms it isn't present locally yet —
this is the local-cache-emptiness fact, not yet a statement about
whether the *remote* pull will succeed.

### 2.3 Reading the `.dvc` pointer — same structure as Day 11, now read from the consumer's side

```yaml
outs:
- md5: 555f037ee350464f52122d087f28e857
  size: 446
  hash: md5
  path: transactions.csv
```

Day 11 covered *producing* this file (`dvc add`); today is the mirror
operation — *consuming* it. The pointer is a precise content
specification: "I expect a file at this path whose MD5 hash is exactly
this value." `dvc pull` has to satisfy that specification, not just
"download something named transactions.csv."

### 2.4 The remote being correctly configured doesn't mean the pull will succeed

```bash
dvc remote list
```
```text
s3    s3://dvc-storage    (default)
```
```ini
[core]
    remote = s3
['remote "s3"']
    url = s3://dvc-storage
    endpointurl = http://localhost:8333
```

Bucket and endpoint are both already correct here — a meaningfully
different starting state than Day 12's lab, where those two fields were
wrong. This matters: don't reflexively re-apply Day 12's fix (bucket/
endpoint corrections) to a problem that's actually a different layer
entirely. Compare against the spec field-by-field (same discipline as
Day 12) before assuming you already know what's broken.

### 2.5 Reproduce the failure and read the error precisely

```bash
dvc pull -v
```
```text
Unable to locate credentials
```

This error is specific and informative: DVC successfully resolved
*which* remote and *which* endpoint to use — it failed specifically at
the **authentication** step, trying to actually talk to the S3 API.

```text
DVC knows the remote?       YES
DVC knows the endpoint?      YES
DVC knows the credentials?   NO   ← this is the actual problem
```

Three fields, three independent facts — confirming the first two are
fine before concluding the third is the missing piece is what correctly
narrows this from "the remote is broken" (Day 12's framing) to "the
remote is reachable but unauthenticated" (today's actual problem).

### 2.6 Setting credentials — and why the *destination file* matters, not just the command

```bash
dvc remote modify s3 access_key_id weedadmin
dvc remote modify s3 secret_access_key weedadmin123
```

```text
dvc remote modify ...          → writes to .dvc/config        (shared, committed)
dvc remote modify --local ...  → writes to .dvc/config.local   (per-machine, gitignored)
```

Both commands are syntactically similar but write to *different files*
— `--local` exists specifically for secrets a team doesn't want
committed to the shared, version-controlled `.dvc/config`. This lab's
grader specifically checks `.dvc/config`, so using `--local` here would
produce working credentials locally while silently failing the actual
verification criterion. Knowing *which* file a config command targets
matters as much as knowing the command itself.

### 2.7 `dvc config --list --show-origin` — proving which file a value actually came from

```bash
dvc config --list --show-origin
```
```text
.dvc/config remote.s3.access_key_id=weedadmin
.dvc/config remote.s3.secret_access_key=weedadmin123
```

`--show-origin` answers exactly the question §2.6 raises: not just
"what's the value," but "which file supplied it." This is the
authoritative way to confirm credentials landed in `.dvc/config` and not
accidentally in `.dvc/config.local` — reading `cat .dvc/config` alone
would show the values too, but wouldn't catch a case where the *same*
key also exists, overriding or alongside, in the local file.

### 2.8 Successful pull — reading the trace, same shape as Day 12's push trace

```bash
dvc pull -v
```
```text
Preparing to transfer data from 's3://dvc-storage/files/md5'
...
A       data/raw/transactions.csv
1 file fetched and 1 file added
```

Mirror image of Day 12's `dvc push -v` trace (`Preparing to transfer
data FROM .dvc/cache TO s3://...`) — here the direction is reversed
(`FROM` the remote), but the same content-addressed `files/md5/...`
layout underlies both operations identically.

### 2.9 Verify by content hash, not just file existence

```bash
ls -lh data/raw/transactions.csv    # confirms existence + size (446 bytes)
md5sum data/raw/transactions.csv    # confirms CONTENT identity
```
```text
555f037ee350464f52122d087f28e857  data/raw/transactions.csv
```

Compare this directly against the `.dvc` pointer's own `md5` field —
matching hashes is categorically stronger proof than "the file exists
and is roughly the right size." A file could exist, be exactly 446
bytes, and still be the *wrong* 446 bytes; the hash match is the actual
content-identity guarantee DVC's whole design is built around (same
"content hash, not filename, is the real identity" principle from
Day 11).

### 2.10 Final confirmation: `dvc status` agrees local state matches metadata

```bash
dvc status
```
```text
Data and pipelines are up to date.
```

This closes the loop opened in §2.2 — the exact command that first
reported `not in cache` now reports full agreement between `.dvc`
metadata and the workspace.

### 2.11 The compressed reasoning chain

```text
Requirement (pull transactions.csv, verify it matches the recorded hash)
   → dvc status                              → "not in cache" — confirms WHAT is missing (§2.2)
   → cat data/raw/transactions.csv.dvc         → the exact expected path + md5 + size
   → dvc remote list / cat .dvc/config          → bucket + endpoint ALREADY correct (unlike Day 12!)
   → dvc pull -v                                → FAILS: "Unable to locate credentials"
   → Diagnose: remote known, endpoint known, credentials MISSING — narrow to exactly this layer
   → dvc remote modify s3 access_key_id ...      (NOT --local — grader checks .dvc/config)
   → dvc remote modify s3 secret_access_key ...
   → dvc config --list --show-origin              → confirm values landed in .dvc/config specifically
   → dvc pull -v                                    → succeeds: "1 file fetched and 1 file added"
   → ls -lh / md5sum data/raw/transactions.csv       → confirm existence AND exact content hash match
   → dvc status                                      → "up to date" — loop closed
```

---

## 3. Concepts (reference)

### 3.1 A fresh clone never includes DVC-tracked data
Only Git-tracked files (source, `.dvc` pointers, config) come along with
`git clone` — the actual large artifacts live exclusively in the DVC
remote and must be explicitly pulled. This isn't a failure mode, it's
the designed-in separation from Days 10–11.

### 3.2 `dvc status`'s "not in cache" vs. "not in remote"
`not in cache` specifically means the local `.dvc/cache/` doesn't have
the object — it says nothing yet about whether the remote has it either.
A `dvc pull` failure afterward could still be for an entirely different
reason (the remote not having it, auth failing, wrong endpoint) —
`dvc status` only narrows the *local* side of the picture.

### 3.3 `.dvc/config` vs. `.dvc/config.local`
Both can hold the same keys; `.dvc/config` is intended to be committed
and shared, `.dvc/config.local` is per-machine and typically gitignored
— the standard place to put real credentials in a team setting, so they
never land in shared, version-controlled config. This specific lab is
an exception: its grader checks the shared file, so `--local` would be
the wrong choice here even though it's usually the *more correct*
general practice.

### 3.4 `dvc config --list --show-origin`
Lists every active DVC config value together with which file it came
from — the authoritative way to confirm a setting landed where you
intended, especially when multiple config files (`config`,
`config.local`, global config) could plausibly supply the same key.

### 3.5 Content-hash verification as the strongest form of "did this work"
File existence and file size are weak signals; a content hash match
(`md5sum` against the `.dvc` pointer's recorded hash) is strong,
specific proof the exact expected bytes are present — the same
principle underlying DVC's entire content-addressable design (Day 11).

### 3.6 `dvc push` and `dvc pull` as mirror operations
Both move objects between the local cache and the configured remote,
along the same `files/md5/<prefix>/<rest>` layout — `push` goes cache →
remote, `pull` goes remote → cache (then materializes into the
workspace). Fixing one direction's configuration generally fixes both,
since they share the same remote config entirely.

---

## 4. Runbook

### 4.1 Enter the project and check status
```bash
cd /root/code/fraud-detection
dvc status
```
```text
data/raw/transactions.csv.dvc:
    changed outs:
        not in cache: data/raw/transactions.csv
```

### 4.2 Inspect the DVC pointer
```bash
cat data/raw/transactions.csv.dvc
```
```yaml
outs:
- md5: 555f037ee350464f52122d087f28e857
  size: 446
  hash: md5
  path: transactions.csv
```

### 4.3 Inspect the remote configuration
```bash
dvc remote list
```
```text
s3    s3://dvc-storage    (default)
```
```bash
cat .dvc/config
```
```ini
[core]
    remote = s3
['remote "s3"']
    url = s3://dvc-storage
    endpointurl = http://localhost:8333
```
Bucket and endpoint are correct — unlike Day 12, this is NOT the
problem here (§2.4).

### 4.4 Reproduce the failure
```bash
dvc pull -v
```
```text
ERROR: ... Unable to locate credentials
```

### 4.5 Configure credentials in the shared config file (not `--local`)
```bash
dvc remote modify s3 access_key_id weedadmin
dvc remote modify s3 secret_access_key weedadmin123
```

### 4.6 Verify the credentials landed in `.dvc/config` specifically
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
dvc config --list --show-origin
```
```text
.dvc/config remote.s3.url=s3://dvc-storage
.dvc/config remote.s3.endpointurl=http://localhost:8333
.dvc/config remote.s3.access_key_id=weedadmin
.dvc/config remote.s3.secret_access_key=weedadmin123
.dvc/config core.remote=s3
```

### 4.7 Pull the dataset
```bash
dvc pull -v
```
```text
Preparing to transfer data from 's3://dvc-storage/files/md5'
...
A       data/raw/transactions.csv
1 file fetched and 1 file added
```

### 4.8 Verify existence and size
```bash
ls -lh data/raw/transactions.csv
```
```text
-rw-r--r-- ... 446 ... data/raw/transactions.csv
```

### 4.9 Verify content hash matches the pointer exactly
```bash
md5sum data/raw/transactions.csv
```
```text
555f037ee350464f52122d087f28e857  data/raw/transactions.csv
```
Matches the `.dvc` pointer's `md5` field exactly.

### 4.10 Final DVC-level confirmation
```bash
dvc status
```
```text
Data and pipelines are up to date.
```

### 4.11 Submit
With `dvc pull` succeeding, the file present with the correct size, the
MD5 matching the pointer exactly, and `dvc status` reporting up to
date, click **Check** in the KodeKloud lab UI.

---

## 5. Final state

| Requirement | Result |
|---|---|
| `dvc status` initially shows `not in cache` | ✅ (expected starting point) |
| `.dvc/config` bucket/endpoint already correct | ✅ (confirmed, not the fix needed) |
| `dvc pull -v` reproduces `Unable to locate credentials` | ✅ |
| `access_key_id`/`secret_access_key` set in `.dvc/config` (not `.local`) | ✅ |
| `dvc pull` succeeds: `1 file fetched and 1 file added` | ✅ |
| File exists, 446 bytes | ✅ |
| MD5 matches `.dvc` pointer exactly | ✅ |
| `dvc status` reports up to date | ✅ |

```text
transactions.csv.dvc (md5: 555f037...e857)
        │
        │ dvc pull
        ▼
   .dvc/config: url + endpoint ✅, credentials ✅ (added)
        │
        ▼
   SeaweedFS: s3://dvc-storage/files/md5/55/5f037...
        │
        ▼
   data/raw/transactions.csv  (446 bytes, md5 matches exactly)
        │
        ▼
   dvc status → "up to date"
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `dvc status` shows `not in cache` right after cloning | Expected — DVC-tracked data is never committed to Git | This is normal; proceed to `dvc pull` |
| `dvc pull -v` fails with `Unable to locate credentials` | `access_key_id`/`secret_access_key` missing from the remote config | `dvc remote modify s3 access_key_id ...` / `secret_access_key ...` |
| Credentials set but the lab still fails verification | Used `--local`, writing to `.dvc/config.local` instead of the checked `.dvc/config` | Re-run `dvc remote modify s3 ...` (no `--local`); confirm with `--show-origin` |
| Re-applied Day 12's bucket/endpoint fix out of habit | Assumed this lab had the same root cause as the previous one without checking | Always diff the current config against the spec first — don't pattern-match a prior fix onto a new symptom |
| `dvc pull` reports success but you're not fully confident the data is right | Only checked file existence/size, not content | `md5sum` the file and compare against the `.dvc` pointer's own `md5` field |
| `dvc config --list --show-origin` shows a value from `config.local` unexpectedly | A credential was set with `--local` at some point, possibly overriding the shared config | Decide deliberately which file should hold it for this task's grading requirement, and remove the other if it conflicts |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.11) out loud
      from "cloned repo is missing transactions.csv" to "MD5-verified
      pull, `dvc status` up to date."
- [ ] Explain, in one sentence, why a freshly-cloned DVC project never
      has the actual tracked data present yet.
- [ ] Explain why `dvc status` reporting `not in cache` doesn't yet tell
      you whether a `dvc pull` will succeed.
- [ ] Explain the difference between `dvc remote modify s3 <key> <val>`
      and the `--local` variant — which file does each write to, and
      why does that distinction matter for this specific lab?
- [ ] Explain why checking the MD5 hash is stronger verification than
      checking the file's existence and size.
- [ ] Explain how you'd distinguish "wrong endpoint/bucket" (Day 12's
      problem) from "missing credentials" (today's problem) purely from
      the `dvc pull -v` error text, without assuming which one it is in
      advance.
- [ ] Break the credentials again (remove them, or add them via
      `--local` instead) and fix it from memory, verifying with
      `--show-origin` each time.

---

## 8. Document template for future lab days

Use this skeleton for `day-N/README.md`:

```markdown
# Day N — <task title>

## 1. Scenario
<what was asked, what was broken/required, constraints>

## 2. Reasoning model — how to derive the fix/commands
<walk the requirement down step by step, in the order you'd actually
discover it: check local state → reproduce the actual failure → read
the error precisely to narrow which layer/field is actually wrong,
rather than assuming it matches a prior similar lab → fix exactly that
layer → verify the fix landed in the RIGHT file/location → verify the
end result by content, not just existence. End with a compressed
arrow-chain version.>

## 3. Concepts (reference)
<one subsection per concept the task exercises — explain WHY, not just WHAT>

## 4. Runbook
<copy-pasteable commands in the order run, with real intermediate
output/results inline where they mattered to the diagnosis>

## 5. Final state
<table + diagram of what the configuration/repository looks like after
completion>

## 6. Troubleshooting
<symptom / cause / fix table>

## 7. Do it yourself (checklist)
<"walk the reasoning chain out loud" + comprehension questions + "break it
again and fix it from memory" prompt>
```
