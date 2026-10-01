# Day 11 — Track a Dataset with DVC

A KodeKloud "100 MLOps" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how Git and DVC actually
divide ownership of a file*.

This is the direct sequel to Day 10 (`dvc init`) — that lab enabled DVC
without tracking anything; this lab performs the actual `dvc add` and
hits the one error that makes the Git/DVC division of labor concrete.

---

## 1. Scenario

The `fraud-detection` repository already has DVC initialized (Day 10).
`data/raw/transactions.csv` is currently tracked directly by **Git**.
Requirement: Git should stop tracking the dataset; **DVC** should take
ownership of it instead, with the resulting DVC metadata committed to
Git.

```text
Before:                          After:
Git                               Git
 └── transactions.csv  ❌          ├── transactions.csv.dvc  ✅
                                   └── .gitignore             ✅
                                            │
                                            └── ignores transactions.csv
                                                     │
                                                     ▼
                                              (managed by DVC)
```

---

## 2. Reasoning model — how to *derive* the fix, not memorize it

### 2.1 Why Git alone is a poor fit for the dataset, specifically

```text
Code (src/, tests/, README)        ML data (data/raw/transactions.csv)
  small, text-based                 can be hundreds of MB–GB, binary
  frequently diffed                  expensive to store in every revision
  naturally suited to Git            not what Git's storage model optimizes for
```

Same root problem as Day 10's `dvc init` motivation, now applied to an
*actual* file instead of discussed abstractly — this lab is where that
theory becomes a concrete migration.

### 2.2 Confirm who currently owns the file before doing anything

```bash
git ls-files data/raw/transactions.csv
```

If Git prints the path, Git is tracking it. This single check is what
tells you *which* migration path applies — a file Git already tracks
needs a different procedure (§2.3–§2.5) than a brand-new, never-tracked
file (which `dvc add` alone would handle).

### 2.3 Why `dvc add` refuses a Git-tracked file — read the error as a feature, not a bug

```bash
dvc add data/raw/transactions.csv
```
```text
ERROR: output 'data/raw/transactions.csv' is already tracked by SCM
```

DVC is explicitly refusing to create an **ambiguous ownership model**:

```text
Git ──────┐
          ├── transactions.csv   ← who actually owns this file's history?
DVC ──────┘
```

If both tools tracked the same file, you'd have two independent,
possibly-diverging version histories for one path — DVC's refusal is a
safety check, not an obstacle to route around. The fix is to resolve
ownership explicitly: Git keeps the *pointer*, DVC takes the *content*.

### 2.4 `git rm --cached` — removing Git's tracking without touching the file

```bash
git rm --cached data/raw/transactions.csv
```

```text
Git index:        remove the file       (this command's actual effect)
Filesystem:        leave it alone         (because of --cached)
```

This is the exact same flag and the exact same reasoning as Day 4
MLOps's `.gitignore`/untracking lab: `--cached` scopes the removal to
Git's index only. Without `--cached`, plain `git rm` deletes the
*working-copy file too* — which would be actively harmful here, since
the next step (`dvc add`) needs the real file's bytes still present on
disk to hash and take ownership of.

```bash
ls -lh data/raw/transactions.csv
```
Confirms the file is still physically present after `git rm --cached` —
worth verifying explicitly, not just trusting the flag did what you
expect.

### 2.5 Now `dvc add` succeeds — and what it actually produces

```bash
dvc add data/raw/transactions.csv
```
```text
data/raw/
├── transactions.csv
├── transactions.csv.dvc
└── .gitignore
```

```text
dvc add
   │
   ▼
Read transactions.csv
   │
   ▼
Calculate a content hash (MD5)
   │
   ▼
Write transactions.csv.dvc   (the pointer/metadata)
   │
   ▼
Write/update .gitignore       (so Git never re-tracks the raw file)
```

### 2.6 Reading a `.dvc` file — it's a pointer, not a copy of the data

```yaml
outs:
- md5: c4dd797a1451653b3cb76a8bb7b2b4d9
  size: 379
  hash: md5
  path: transactions.csv
```

```text
outs:  →  "this file describes a tracked OUTPUT/artifact"
path:  →  which file this metadata refers to
size:  →  379 bytes — useful for a basic sanity check
md5:   →  a content fingerprint, not an arbitrary ID
```

A `.dvc` file is tiny (a handful of lines) regardless of whether the
dataset it describes is 379 bytes or 379 GB — this is precisely why
committing it to Git is cheap even for enormous datasets: Git only ever
stores this small pointer, never the dataset itself.

### 2.7 Why content hashing (not just a filename) is the actual identity mechanism

```text
transactions.csv  →  md5()  →  c4dd797a...   (version A)
   (one row edited)
transactions.csv  →  md5()  →  a9f821bb...   (version B — different hash)
```

**The identity of the data is tied to its contents, not its filename.**
Two files named identically but containing different bytes get
different hashes — this is how DVC (and, later, a DVC remote) can
distinguish dataset versions reliably, the same principle behind content-
addressed storage generally (and conceptually similar to how Git itself
identifies commits/blobs by content hash, not by name).

### 2.8 Why DVC writes a `.gitignore`, not just the `.dvc` pointer

```text
data/raw/.gitignore
    /transactions.csv
```

Without this, a careless future `git add .` could silently re-track the
raw dataset directly into Git — re-creating the exact ambiguous-
ownership problem `dvc add` just resolved (§2.3). The `.gitignore` entry
is DVC actively defending the ownership boundary it just established,
not an incidental side effect.

### 2.9 Reading the staged diff correctly — a "D" here is expected, not alarming

```bash
git status --short
```
```text
A  data/raw/.gitignore
D  data/raw/transactions.csv
A  data/raw/transactions.csv.dvc
```

```text
D transactions.csv       → Git is being told to stop tracking it
                            (NOT "the file was deleted from disk")
A transactions.csv.dvc   → Git starts tracking the new pointer instead
A .gitignore              → Git starts tracking the protective ignore rule
```

Seeing a `D` here could look alarming out of context — reading it
correctly as "Git's *tracking* of this path is being removed" rather
than "this file is gone" is exactly the distinction §2.4 sets up.

### 2.10 Why the `.dvc` file belongs in Git history, not just on disk

```text
Git commit A  →  transactions.csv.dvc  →  hash AAA
Git commit B  →  transactions.csv.dvc  →  hash BBB   (dataset changed)
```

Because the `.dvc` pointer is small and text-based, Git can diff and
version *it* efficiently even though it can't efficiently version the
actual dataset — this is what lets a specific Git commit answer "which
exact dataset version was used here," reconstructing reproducibility
without ever putting the heavy artifact into Git itself.

### 2.11 Stage exactly what DVC tells you to, then commit

```bash
git add data/raw/transactions.csv.dvc data/raw/.gitignore
git commit -m "Track transactions dataset with DVC"
```

`dvc add`'s own output literally names the files to `git add` — trust
that output rather than guessing. After the commit, `git status --short`
returning nothing and `git ls-files data/raw/transactions.csv` returning
nothing are the two independent facts that confirm the migration is
complete: Git doesn't track the raw file anymore, and the DVC pointer is
now a real, committed part of history.

### 2.12 The compressed reasoning chain

```text
Requirement (stop Git tracking the dataset; DVC takes ownership; committed)
   → Confirm Git currently owns it: git ls-files data/raw/transactions.csv
   → Attempt dvc add directly                      → FAILS: "already tracked by SCM"
   → Diagnose: ambiguous ownership is the actual reason for the refusal
   → git rm --cached data/raw/transactions.csv       (index only — file stays on disk)
   → Verify file still exists: ls -lh ...
   → dvc add data/raw/transactions.csv               → creates .dvc pointer + .gitignore
   → Inspect the .dvc file: md5/size/path — a POINTER, not a copy
   → git status --short                               → A .gitignore / D csv / A csv.dvc (all expected)
   → git add transactions.csv.dvc .gitignore
   → git commit -m "Track transactions dataset with DVC"
   → Verify: git status clean; git ls-files <path> returns NOTHING; file still exists on disk
```

---

## 3. Concepts (reference)

### 3.1 `dvc add` vs. the Day 10 `dvc init`
`dvc init` enables DVC project-wide infrastructure and tracks nothing.
`dvc add <path>` is the operation that actually starts versioning one
specific file/directory — today's lab performs the second step the
prior lab deliberately stopped short of.

### 3.2 `git rm --cached` (recap from Day 4 MLOps)
Removes a path from Git's index/tracking while leaving the working-copy
file untouched — essential whenever "stop Git from tracking this" must
NOT mean "delete this file," which is exactly the case when migrating
ownership to another tool that still needs the file's bytes present.

### 3.3 Content hashing as identity
A file's DVC-tracked identity is a hash of its actual bytes (MD5 here),
not its path or filename — two files with identical names but different
content are different versions; the same file moved/renamed but with
identical content hashes the same.

### 3.4 `.dvc` files are pointers, never the data itself
Tiny, text-based, Git-friendly metadata files — size and content are
independent of how large the actual tracked artifact is. This is the
mechanism that lets Git stay small while still versioning references to
arbitrarily large datasets.

### 3.5 `.gitignore` as an active ownership boundary, not passive cleanup
When DVC writes a `.gitignore` entry for a file it now tracks, it's
preventing a *specific* future mistake (accidental re-tracking by Git),
not just tidying up — the same "config as defense, not decoration"
framing used for `.dvc/.gitignore` in Day 10.

### 3.6 DVC remotes (context for where this leads next)
A `.dvc` pointer committed to Git identifies a dataset version, but the
actual bytes still have to live somewhere retrievable — a DVC remote
(S3, GCS, Azure Blob, SSH storage, etc.) is that storage, analogous to a
Git remote (GitHub/GitLab) but for the data DVC manages instead of Git's
own objects.

```text
git push   →  Git commits/branches/metadata   →  Git remote
dvc push   →  datasets/models/artifacts        →  DVC remote
```

### 3.7 Reproducibility: code version + data version, together
```text
Git commit  →  .dvc metadata  →  dataset hash  →  exact dataset version
```
Versioning only source code is insufficient for ML specifically, because
identical code trained against a different dataset can produce a
meaningfully different model — the whole point of this lab's migration
is making the dataset's exact version as reconstructible from history as
the code's always was.

---

## 4. Runbook

### 4.1 Confirm current Git ownership
```bash
cd /root/code/fraud-detection
git ls-files data/raw/transactions.csv
```
```text
data/raw/transactions.csv
```
Confirms Git currently tracks it.

### 4.2 Attempt `dvc add` directly — reproduce the expected failure
```bash
dvc add data/raw/transactions.csv
```
```text
ERROR: output 'data/raw/transactions.csv' is already tracked by SCM
```

### 4.3 Untrack from Git's index only — keep the file on disk
```bash
git rm --cached data/raw/transactions.csv
```
```text
rm 'data/raw/transactions.csv'
```
```bash
ls -lh data/raw/transactions.csv
```
Confirms the file still physically exists.

### 4.4 Add it to DVC
```bash
dvc add data/raw/transactions.csv
```
```text
To track the changes with git, run:

        git add data/raw/transactions.csv.dvc data/raw/.gitignore
```

### 4.5 Inspect what DVC generated
```bash
cat data/raw/transactions.csv.dvc
```
```yaml
outs:
- md5: c4dd797a1451653b3cb76a8bb7b2b4d9
  size: 379
  hash: md5
  path: transactions.csv
```
```bash
cat data/raw/.gitignore
```
```text
/transactions.csv
```

### 4.6 Review the Git status before staging
```bash
git status --short
```
```text
A  data/raw/.gitignore
D  data/raw/transactions.csv
A  data/raw/transactions.csv.dvc
```

### 4.7 Stage exactly what DVC told you to
```bash
git add data/raw/transactions.csv.dvc data/raw/.gitignore
```

### 4.8 Commit
```bash
git commit -m "Track transactions dataset with DVC"
```
```text
[master c1a6981] Track transactions dataset with DVC
```

### 4.9 Final verification
```bash
git status --short
```
(no output — clean)
```bash
git ls-files data/raw/transactions.csv
```
(no output — Git no longer tracks the raw file)
```bash
test -f data/raw/transactions.csv && echo "Dataset exists"
```
```text
Dataset exists
```

### 4.10 Click "Check" in the lab UI to validate.

---

## 5. Final state

| Requirement | Result |
|---|---|
| Git no longer tracks `transactions.csv` | ✅ `git ls-files` returns nothing |
| Dataset file still present on disk | ✅ |
| `transactions.csv.dvc` created and committed | ✅ |
| `.gitignore` protecting the raw file created and committed | ✅ |
| Working tree clean after commit | ✅ |

```text
                 Git
                  │
        ┌─────────┴───────────┐
        ▼                     ▼
 .gitignore            transactions.csv.dvc
                              │
                              │ identifies (by content hash)
                              ▼
                    transactions.csv
                     (on disk, DVC-managed,
                      no longer Git-tracked)
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `ERROR: output '...' is already tracked by SCM` | Git still tracks the file at the index level | `git rm --cached <file>` first, then retry `dvc add` |
| Dataset file disappeared after trying to untrack it | Used plain `git rm` instead of `git rm --cached` | Restore from a previous commit (`git checkout HEAD~1 -- <path>`) or regenerate the data; re-run with `--cached` |
| `dvc add` succeeds but `git status` shows nothing new | Forgot to `git add` the `.dvc` file and `.gitignore` after `dvc add` | `git add <path>.dvc <path's dir>/.gitignore` — DVC's own output tells you exactly which files to add |
| Confused by a `D` next to the dataset in `git status` | Misread "Git stops tracking" as "file deleted" | `D` here only reflects the Git index change; verify with `ls -lh` that the file is untouched on disk |
| `git ls-files <path>` still returns the dataset path after the commit | The `.dvc`/`.gitignore` changes were never actually committed, or the wrong path was staged | Re-check `git status`; confirm the commit actually happened (`git log --oneline`) |
| Unsure whether two versions of a dataset are "the same" | Comparing filenames instead of content | Compare the `md5` hash inside each version's `.dvc` file — that's the actual identity, not the filename |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.12) out loud
      from "Git is tracking this dataset, DVC should instead" to
      "verified: untracked by Git, still on disk, DVC metadata
      committed."
- [ ] Explain, in one sentence, why `dvc add` refuses a file Git is
      already tracking, rather than just taking it over silently.
- [ ] Explain why `git rm --cached` is the correct command here, and
      what would have gone wrong with plain `git rm`.
- [ ] Explain what a `.dvc` file actually contains, and why its size is
      unrelated to the size of the dataset it describes.
- [ ] Explain why DVC identifies a dataset version by content hash
      rather than by filename.
- [ ] Repeat the full migration on a different file from memory,
      verifying with both `git ls-files` (should return nothing) and
      `test -f` (should confirm the file still exists) — not just one
      or the other.

---

## 8. Document template for future lab days

Use this skeleton for `day-N/README.md`:

```markdown
# Day N — <task title>

## 1. Scenario
<what was asked, what was broken/required, constraints>

## 2. Reasoning model — how to derive the fix/commands
<walk the requirement down step by step, in the order you'd actually
discover it: establish current ownership/state → reproduce the expected
failure → diagnose why it's a protective check, not an obstacle → fix →
verify from multiple independent angles. End with a compressed
arrow-chain version.>

## 3. Concepts (reference)
<one subsection per concept the task exercises — explain WHY, not just WHAT>

## 4. Runbook
<copy-pasteable commands in the order run, with real intermediate
output/results inline where they mattered to the diagnosis>

## 5. Final state
<table + diagram of what the repository looks like after completion>

## 6. Troubleshooting
<symptom / cause / fix table>

## 7. Do it yourself (checklist)
<"walk the reasoning chain out loud" + comprehension questions + "redo
from memory, verifying every step" prompt>
```
