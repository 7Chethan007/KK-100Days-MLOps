# Day 18 — Version Datasets and Models Across Git Branches

A KodeKloud "100 MLOps" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive it yourself*.

Day 17 established the distinction between a disposable DVC
*experiment* and a deliberate Git *commit* that promotes one. This lab
is the long-lived version of that same idea: instead of comparing
throwaway experiments on one branch, it uses Git branches and tags to
keep two *permanent*, independently-restorable versions of the same
dataset/pipeline/model triple side by side.

---

## 1. Scenario

Repository: `/root/code/fraud-detection/`. Starting state: `main`,
clean working tree, an existing DVC-tracked pipeline
(`process_data` → `split_data` → `train`) with a baseline raw dataset.

Required steps:

1. Tag the baseline as `v1.0`.
2. Create branch `v2-improved`.
3. Swap in a staged, larger dataset (`transactions_v2.csv`) as the
   active input.
4. Retrack it with DVC and retrain the pipeline.
5. Commit the updated metadata and model version on `v2-improved`.
6. Switch back to `main` and restore the original dataset and model.

```text
main (v1.0 tag)                    v2-improved
  │                                      │
  ├── transactions.csv (379B)            │
  ├── model.pkl (hash 331af74a...)       │
  │                                      │
  └──────────────── git switch -c ───────┤
                                          ├── cp transactions_v2.csv → transactions.csv
                                          ├── dvc add + dvc repro
                                          ├── model.pkl (hash 34088e7a...)
                                          └── git commit "Version improved dataset..."
  ◄── git switch main ─────────────────────┘
  │
  └── dvc checkout  (restores v1 artifacts on disk)
```

---

## 2. Reasoning model — how to *derive* the commands, not memorize them

### 2.1 Git and DVC version two different things, and neither substitutes for the other

```text
Git tracks:  code, dvc.yaml, dvc.lock, *.dvc metadata  → small, diffable, in every commit
DVC tracks:  actual dataset/model bytes                → large, content-addressed, in the DVC cache
```

Switching Git branches only changes which *metadata* is checked out —
which hash each `.dvc` file points to. It does **not**, by itself,
materialize the matching dataset/model bytes on disk; that is what
`dvc checkout` does, separately, by reading the now-current metadata
and pulling the matching content from the cache. This is the same
"Git commit vs. workspace vs. experiment" three-way split from Day 17,
just applied across long-lived branches instead of short-lived
experiments.

### 2.2 Why tag *before* branching, not after

```text
git tag -a v1.0 -m "..."   → pins a permanent, named pointer to the current commit
git switch -c v2-improved  → branch HEAD can move forward freely from here
```

A branch pointer moves every time you commit on it; a tag does not
move once created. Tagging the baseline *before* creating the
experimental branch guarantees `v1.0` stays a stable reference to
"dataset+model as they were before any v2 change," independent of
whatever `v2-improved` does afterward — you could delete
`v2-improved` entirely and `v1.0` would still resolve correctly.

### 2.3 Replacing the active input file, not the staged source

```bash
cp data/raw/transactions_v2.csv data/raw/transactions.csv
```

The pipeline's `process_data` stage is wired to the path
`data/raw/transactions.csv` (set up long before this lab — see Day 14's
`dvc.yaml` dependency declarations). Renaming or moving
`transactions_v2.csv` into that path would destroy the one copy of the
"staged, not-yet-active" file; copying preserves `transactions_v2.csv`
as a reusable source while still landing new content at the exact path
the pipeline already depends on. This is the same "the pipeline keys
on exact file paths, not content similarity" mechanic from Day 14 — the
DAG doesn't know or care that the new file is "an improved version" of
anything; it only reacts to the file at `data/raw/transactions.csv`
having a different hash than before.

### 2.4 `dvc add` recalculates the hash; it doesn't move or validate the data semantically

```bash
dvc add data/raw/transactions.csv
```

`dvc add` hashes the current file contents and rewrites
`transactions.csv.dvc` to point at that new hash — it has no concept
of "is this actually an improvement," only "here is what's on disk
right now." The resulting changed `.dvc` file is exactly what makes
`process_data`'s dependency look stale to `dvc status`, which is the
same dependency-hash mechanism from Day 14/15 that governs whether
`dvc repro` thinks a stage needs to rerun.

### 2.5 `dvc repro` reruns from the first changed dependency forward — same DAG rule as Day 14

```text
process_data ← raw dataset changed → reruns
split_data   ← its input (clean_transactions.csv) just changed → reruns
train        ← its input (split datasets) just changed → reruns
```

None of the three stages needed to be told individually to rerun —
`dvc repro` walks the DAG and reruns every stage whose declared
dependency hash differs from what `dvc.lock` last recorded, cascading
forward exactly as in Day 14. The model's hash changing
(`331af74a...` → `34088e7a...`) is a *consequence* of the raw dataset
changing, not something committed separately or by hand.

### 2.6 Commit the `.dvc` file **and** `dvc.lock` together — a split commit leaves an inconsistent branch state

```text
transactions.csv.dvc  → "this is the raw dataset version this branch uses"
dvc.lock               → "this is what every stage actually produced from it,
                           including models/model.pkl's hash"
```

Committing only one of these would leave `v2-improved` in a state
where Git says one thing about the input and the lockfile says
something else about the pipeline's recorded output — exactly the
kind of drift Day 15/16's `params:`/`metrics:` discipline was designed
to prevent by keeping declared intent and recorded result in the same
commit.

### 2.7 Switching branches restores metadata instantly; it does not restore artifact bytes

```text
git switch main      → .dvc files and dvc.lock now say "v1 hashes" (instant, Git-only)
<on disk right now>   → may still physically be the v2-improved files (stale until told otherwise)
dvc checkout          → reads the now-current (v1) metadata, pulls matching bytes from DVC cache
```

This is the crux of the lab and the reason `dvc checkout` exists as a
distinct command from `git checkout`/`git switch`: Git's checkout only
ever touches files Git itself tracks (code, `.dvc` metadata,
`dvc.lock`), and the actual dataset/model files are deliberately
*not* Git-tracked (they're too large / binary / content-addressed
instead). Skipping `dvc checkout` after switching branches is the
single most likely way to end up with "metadata says v1, but the file
on disk is still v2."

### 2.8 Verifying restoration means checking the DVC hash, not just the branch name

```text
git branch --show-current  → "main"          (necessary, not sufficient)
dvc status                 → "up to date"     (confirms disk matches checked-out metadata)
cat transactions.csv.dvc   → hash == baseline  (confirms it's specifically the ORIGINAL version)
```

Being on `main` only proves you're looking at the right *metadata*; it
doesn't by itself prove the right *content* is on disk until `dvc
status` and the recorded hash both confirm it — the same "don't trust
the request accepted, verify the actual resulting state" discipline
carried over from the AWS side of this series (EBS/EC2/EIP "response
accepted ≠ resource ready").

### 2.9 The compressed reasoning chain

```text
Requirement: version dataset+model across branches, restore reliably
   → tag the known-good baseline FIRST, before branching (§2.2)
   → branch for the experiment, so main stays untouched
   → cp (not mv) the staged dataset into the pipeline's exact input path (§2.3)
   → dvc add → new content hash recorded in .dvc (§2.4)
   → dvc repro → DAG cascades the rerun forward from the changed dependency (§2.5)
   → commit .dvc AND dvc.lock together — they describe one consistent state (§2.6)
   → git switch main → metadata flips instantly, disk content does NOT (§2.7)
   → dvc checkout → materializes the matching baseline bytes from cache
   → verify via dvc status + the actual recorded hash, not just the branch name (§2.8)
```

---

## 3. Concepts (reference)

### 3.1 Git tag vs. Git branch
A tag is an immutable pointer to one specific commit — useful for
marking a release or baseline that should never silently move. A
branch is a mutable pointer that advances with each new commit on it —
useful for isolating ongoing, uncertain work from a stable reference
point.

### 3.2 Why DVC-tracked files aren't Git-tracked directly
Datasets and models are typically large, binary, and change often in
ways Git's text-diffing isn't designed for. DVC stores the actual
content in a local cache (and optionally a remote), keyed by content
hash, while Git stores only the small `.dvc` metadata file pointing at
that hash — keeping the Git repository itself small and diffable.

### 3.3 `git switch`/`checkout` vs. `dvc checkout`
Git's checkout operates only on files Git tracks. `dvc checkout`
operates only on files DVC tracks, using whatever `.dvc`/`dvc.lock`
metadata is *currently* checked out by Git to decide which cached
content to restore. The two commands are complementary, not
redundant — switching branches without a follow-up `dvc checkout`
leaves DVC-tracked content stale relative to the newly-checked-out
metadata.

### 3.4 Content hash as the version identity
A DVC hash is derived from file contents. Two files with the same
hash are guaranteed to have identical content; a different hash after
replacing a "same-named" file is simply confirming that the file's
actual bytes changed, which is exactly what triggers `dvc status` to
flag the dependency as modified.

### 3.5 Why both `.dvc` metadata and `dvc.lock` matter
The `.dvc` file for a raw input and the pipeline's `dvc.lock` describe
two different things: what version of an input exists, and what the
pipeline actually produced when it last ran against some input.
Committing them together keeps "what we fed the pipeline" and "what
the pipeline reported it produced" from drifting apart on the same
branch.

### 3.6 Reproducibility as the end goal
The entire exercise — tag, branch, replace, retrain, commit, switch
back, restore — exists to answer one operational question reliably:
"can I get back to exactly the baseline dataset and model, on demand,
without manual bookkeeping?" Verifying the restored hash against the
originally recorded baseline hash is the direct test of that
guarantee.

---

## 4. Runbook

### 4.1 Confirm starting state
```bash
cd /root/code/fraud-detection
git status --short --branch
git branch --show-current
git tag --list
```
Expected: branch `main`, clean working tree, no existing tags.

### 4.2 Inspect baseline DVC metadata
```bash
cat data/raw/transactions.csv.dvc
grep -A12 -B2 -E 'transactions.csv|model.pkl' dvc.lock
dvc status
```
```text
Data and pipelines are up to date.
```

Baseline hashes recorded in the lab:

| Artifact | v1 DVC hash | Size |
|---|---|---|
| `data/raw/transactions.csv` | `c4dd797a1451653b3cb76a8bb7b2b4d9` | 379 bytes |
| `data/processed/clean_transactions.csv` | `28d3a228879683fe66e456ee452bf87c` | 371 bytes |
| `models/model.pkl` | `331af74a93fb4992f3e5b81b98d5e52f` | 67,737 bytes |

### 4.3 Tag the baseline and create the experiment branch
```bash
git tag -a v1.0 -m "Baseline dataset and model v1.0"
git show v1.0 --stat
git tag --list
git switch -c v2-improved
```
`v1.0` resolved to commit `60221edd9eb65b3f96718db34032865ddbdbd0fc`.

### 4.4 Inspect the improved dataset and pipeline
```bash
ls -lh data/raw/transactions.csv data/raw/transactions_v2.csv
cat dvc.yaml
```
`transactions.csv` is 379 bytes; the staged `transactions_v2.csv` is
723 bytes. Do not delete `transactions_v2.csv` — it is the staged
source, not a temp file.

### 4.5 Replace and retrack the dataset
```bash
cp data/raw/transactions_v2.csv data/raw/transactions.csv
dvc add data/raw/transactions.csv
cat data/raw/transactions.csv.dvc
dvc status
```
New recorded hash: `bd6bde1f4b10d12c6249a2d021b57e87`. `dvc status`
correctly reported `process_data`'s raw-dataset dependency as modified.

### 4.6 Retrain the pipeline
```bash
dvc repro
```
```text
Trained model on 16 rows
```
Executed stages, in order: `process_data` → `split_data` → `train`.

```bash
dvc status
```
```text
Data and pipelines are up to date.
```

Verify the new model hash:
```bash
grep -A8 -B3 'models/model.pkl' dvc.lock
```
New hash: `34088e7af3323ec5e2918cafe5438a4e` (baseline was
`331af74a93fb4992f3e5b81b98d5e52f`).

### 4.7 Commit the v2 version
```bash
git status --short
```
```text
 M data/raw/transactions.csv.dvc
 M dvc.lock
```
```bash
git add data/raw/transactions.csv.dvc dvc.lock
git commit -m "Version improved dataset and retrained model"
git status --short --branch
git log -1 --oneline
```
```text
0c28eef Version improved dataset and retrained model
```
Working tree clean on `v2-improved`.

### 4.8 Restore the baseline on `main`
```bash
git switch main
dvc checkout
```

### 4.9 Verify restoration
```bash
dvc status
grep -A3 -B1 'models/model.pkl' dvc.lock
cat data/raw/transactions.csv.dvc
```

| Check | Expected baseline | Result |
|---|---|---|
| Current branch | `main` | Passed |
| DVC status | Up to date | Passed |
| Raw dataset hash | `c4dd797a1451653b3cb76a8bb7b2b4d9` | Match |
| Model hash | `331af74a93fb4992f3e5b81b98d5e52f` | Match |

Additional Git-level cross-check:
```bash
git status --short --branch
git show v1.0:dvc.lock
git log --oneline --decorate --all -5
```

---

## 5. Final state

```text
main (tag: v1.0)                          v2-improved
  transactions.csv hash: c4dd797a...        transactions.csv hash: bd6bde1f...
  model.pkl hash:        331af74a...        model.pkl hash:        34088e7a...
  (restored via dvc checkout)               (committed 0c28eef)
```

| Requirement | Result |
|---|---|
| `v1.0` tag created on the original baseline commit | ✅ |
| `v2-improved` branch created from that baseline | ✅ |
| Dataset swapped and retracked (`dvc add`) on `v2-improved` | ✅ |
| Pipeline retrained (`dvc repro`), model hash changed | ✅ |
| `.dvc` + `dvc.lock` committed together on `v2-improved` | ✅ |
| `main` restored to original dataset + model hash via `dvc checkout` | ✅ |

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `dvc status` reports a changed dependency | Data or code differs from the last recorded state | Inspect the dependency hash; run `dvc repro` if the change is intentional |
| `dvc add` reports changes to track with Git | The `.dvc` metadata itself changed | Stage and commit the `.dvc` file |
| `dvc repro` reruns a stage repeatedly even when `dvc status` says up to date | DVC still considers the stage runnable for some reason | Inspect `dvc.yaml`, `dvc.lock`, and `dvc repro -v` for the actual trigger |
| Model stays at the wrong version after switching branches | Branch switch only updated Git metadata — artifact on disk wasn't refreshed | Run `dvc checkout`, then re-verify with `dvc status` |
| `dvc checkout` can't restore an artifact | Required content missing from the local cache or remote | Check cache/remote availability before further recovery attempts |
| Working tree has unexpected modifications before a commit | Metadata or tracked files changed mid-workflow | Run `git status --short` before every commit, not just at the end |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.9) out loud
      from "tag the baseline" to "verify the restored hash on `main`."
- [ ] Explain, in one sentence, why `git switch main` alone does not
      guarantee the baseline model file is actually on disk.
- [ ] Explain why the dataset was copied (`cp`) into
      `transactions.csv` rather than renaming
      `transactions_v2.csv` directly.
- [ ] Explain why `.dvc` metadata and `dvc.lock` were committed in the
      same commit rather than separately.
- [ ] Explain the difference between what `git tag` and `git branch`
      each guarantee about whether a reference can move.
- [ ] From memory, create a third branch (`v3-experimental`) from the
      `v1.0` tag (not from `main`'s current tip), swap in a different
      dataset, retrain, and restore back to `v1.0` without copying any
      command from this file.

---

## 8. Document template for future lab days

Use this skeleton for `day-N/README.md`:

```markdown
# Day N — <task title>

## 1. Scenario
<what was asked, which account/region/resources, constraints, credentials
if relevant>

## 2. Reasoning model — how to derive the commands
<walk the requirement down to a subsystem/resource hierarchy, step by step,
in the order you'd actually discover it: "what resource, is it global or
region-scoped, what existing state constrains my choice, what CLI
service+operation performs the read (exact lookup vs. filtered search),
what performs the write, how do I verify the SPECIFIC field the
requirement names." End with a compressed step-chain.>

## 3. Concepts (reference)
<one subsection per concept the task exercises — explain WHY, not just WHAT>

## 4. Runbook
<copy-pasteable commands in the order run, WITH the actual intermediate
output/results captured inline as code blocks, not just the commands>

## 5. Final state
<table + diagram of what the infrastructure looks like after completion>

## 6. Troubleshooting
<symptom / cause / fix table>

## 7. Do it yourself (checklist)
<"walk the reasoning chain out loud" + comprehension questions +
"redo without copy-pasting, recompute derived values" prompt>
```
