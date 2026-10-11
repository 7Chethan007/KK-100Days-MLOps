# Day 19 — Production DVC Pipeline with SeaweedFS Remote

A KodeKloud "100 MLOps" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive it yourself*.

Day 18 ended with a complete dataset/model/release story on a single,
already-correct pipeline. This lab starts from a *partially broken*
one — a stage whose script writes one filename while the pipeline
declares another — and adds a fourth layer on top of Day 18's
Git+DVC pair: a remote artifact store (SeaweedFS, via DVC's S3-
compatible interface), which was first introduced conceptually in Day
13's `dvc push`/`dvc pull` arc.

---

## 1. Scenario

Repository: `/root/code/ml-pipeline/`. Starting state: three DVC
stages already declared, two scripts staged under `scripts-staging/`,
and a DVC remote preconfigured against a SeaweedFS S3-compatible
endpoint.

Required outcome: a correctly defined five-stage pipeline
(`ingest` → `validate` → `preprocess` → `train` → `evaluate`) that
runs end to end, pushes its cacheable artifacts to the remote, and is
captured as a tagged Git release.

```text
data/raw/data.csv
      │
      ├──────────────┐
      ▼              ▼
  1. ingest      2. validate ──→ reports/validation.json
      │
      ▼
  3. preprocess ──→ data/processed/clean.csv
      │
      ▼
  4. train ──→ models/model.pkl, data/processed/test_split.csv, metrics.json
      │
      ▼
  5. evaluate ──→ reports/evaluation.json
```

Final result: commit `525e29f`, tag `v1.0`, `dvc push` → 3 files,
`dvc status` → up to date.

---

## 2. Reasoning model — how to *derive* the commands, not memorize them

### 2.1 Four separable problems, not one

```text
Pipeline correctness   → every stage runs, produces declared outputs
Dependency tracking    → DVC understands data/params/script relationships
Artifact storage       → dvc push lands cache objects on the remote
Release management     → git commit + tag identifies that project state
```

A successful `dvc repro` proves the first. It proves nothing about the
second two. This is the same principle as Day 18's §2.1 split between
Git-tracked metadata and DVC-tracked content — different tools solving
different problems, with success in one layer never implying success
in another. Treat each as independently verifiable, not as a single
pass/fail.

### 2.2 `ingest` and `validate` sharing a dependency is a DAG fact, not an assumption

```text
data/raw/data.csv ──┬──→ ingest
                     └──→ validate
```

Nothing about a five-stage pipeline *implies* a strictly linear chain.
The actual dependency graph is only ever what `dvc.yaml` declares —
the same "exact path matching, not inferred relationships" mechanic
from Day 14/18 that governs which stages `dvc repro` considers stale.
Two stages can legitimately share one upstream dependency instead of
chaining through each other.

### 2.3 An S3-compatible URL doesn't tell you which storage backend is behind it

```text
dvc remote list        →  s3   s3://dvc-storage   (default)
dvc config --list       →  remote.s3.endpointurl=http://localhost:8333
```

`s3://` identifies the *protocol* DVC speaks, not the *provider*
serving it — plenty of non-AWS object stores (SeaweedFS, MinIO, Ceph)
expose an S3-compatible API on their own endpoint. `dvc remote list`
alone under-determines the backend; `dvc config --list` is the command
that actually resolves which service is listening, by surfacing
`endpointurl`. This is the DVC-remote analogue of Day 11's
"`AttachmentCount` alone doesn't tell you who" — the summary view
hints at a fact, but the decisive check is one level deeper.

### 2.4 Configuration tells you *intent*; a successful push is the only *operational* proof

```text
dvc config --list   → "DVC intends to talk to http://localhost:8333"
dvc push             → "3 files pushed"  (proof the connection, auth, and upload path all work)
```

Reading a config file only tells you what *should* happen. The
equivalent of the Cloud-AWS series' "response accepted ≠ resource
ready" pattern applies here too: don't treat a correct-looking config
as proof of a working remote — run the operation and observe the
actual result count.

### 2.5 A script's print statement and a pipeline's declared output are two independent claims

```text
script prints:        "Preprocessed: 20 clean rows"     ← the script's own claim
dvc.yaml declares:    outs: [data/processed/cleaned.csv] ← DVC's expectation
actual file written:  data/processed/clean.csv            ← what really exists
```

The failure (`output 'data/processed/cleaned.csv' does not exist`)
occurred because the script's actual write path and the pipeline's
declared output path disagreed — `clean.csv` vs. `cleaned.csv`. A
script can be entirely successful on its own terms and the pipeline
can still fail, because DVC validates against its *declared* contract,
not against whatever the script happens to believe "success" means.
This is the same category of lesson as Day 19 DevOps's `appsv` typo —
a command can succeed while quietly pointing at the wrong path, and
the mismatch only surfaces at the consuming layer.

### 2.6 Diagnosing a contract mismatch: fix the mismatch, not just one side of it blindly

```text
1. Read the full error — exact stage, exact missing output
2. Inspect the stage's script — what path does it actually write?
3. Compare against dvc.yaml's declared outs — do they agree?
4. Decide which side is "correct" (here: align with what train.py already expects)
5. Fix the mismatch, rerun
```

Either side could have been edited — change the script to write
`cleaned.csv`, or change `dvc.yaml` to expect `clean.csv`. The decision
here wasn't arbitrary: `clean.csv` was chosen because the
already-staged `train.py` was written to consume that exact filename,
so aligning the earlier stage's output to the later stage's
expectation avoided a second, needless edit.

### 2.7 `cp` vs. `mv` for staged files — preserving the checker's expected state

```text
mv scripts-staging/train.py scripts/train.py   → scripts-staging/train.py now MISSING
cp scripts-staging/train.py scripts/train.py   → exists in BOTH locations
```

The lab's acceptance checks validated `scripts-staging/` as well as
`scripts/`. `mv` satisfies "the pipeline has the script it needs" while
silently breaking a separate, unstated requirement that the original
staged file still be present. The general lesson: before relocating or
deleting a pre-staged input, check whether anything downstream — a lab
checker, a teammate's script, a later pipeline stage — still expects
the original path to exist.

### 2.8 Declaring `params` is what lets DVC detect a training-relevant change

```text
train:
  params: [n_estimators, max_depth, test_size, random_seed]
```

Declaring these in `dvc.yaml` (not just reading them inside
`train.py`) is what makes `dvc status`/`dvc repro` treat a change to
`params.yaml` as a reason to rerun `train` — identical in spirit to
Day 15's `params:` reclassification: a dependency DVC doesn't know
about is a dependency it can't react to, no matter how real the
coupling is in the actual code.

### 2.9 `cache: false` on metrics/reports — a metric is meant to be read, not just cached

```text
metrics:
  - metrics.json:
      cache: false
outs:
  - reports/evaluation.json:
      cache: false
```

`models/model.pkl` is a regular cached output — it's large, binary,
and exactly what needs content-addressed versioning and remote
storage. `metrics.json`/`evaluation.json` are small, human-readable,
and primarily meant to be diffed and displayed (`dvc metrics show`,
`dvc metrics diff`, same tools from Day 16) — `cache: false` keeps
them out of the DVC cache/push cycle while still having DVC track them
as declared outputs.

### 2.10 Metrics of `1.0` on four test samples is a lab-scale artifact, not a production claim

```text
test_samples: 4, accuracy/f1/precision/recall: all 1.0
```

A perfect score on four held-out rows is exactly what you'd expect
from an undersized test split, not evidence of a production-ready
fraud model — the same caution as Day 17's F1-over-accuracy framing
for imbalanced data, taken one step further: even the "better" metric
is meaningless at this sample size. `dvc repro` succeeding and
`dvc status` reporting "up to date" confirm pipeline *consistency*,
not model *quality* — two different questions entirely.

### 2.11 Push → commit → tag, strictly in that order

```text
dvc push    → artifacts must exist remotely before the release claims they do
git commit  → captures the project state (code, dvc.yaml, dvc.lock, metrics, reports)
git tag     → names that already-committed state as a release
```

Tagging before committing would point `v1.0` at an incomplete state.
Committing before pushing would create a release commit whose
referenced artifacts aren't actually retrievable from the remote yet.
Each command's effect is strictly additive on the previous one — this
mirrors Day 18's "tag before branching" ordering lesson, just with one
more layer (the remote) added at the front.

### 2.12 The compressed reasoning chain

```text
Requirement: correct 5-stage pipeline, pushed to SeaweedFS, tagged release
   → diagnose remote backend via dvc config --list, not just dvc remote list (§2.3)
   → fix the preprocess script/dvc.yaml filename mismatch, align to train.py's expectation (§2.5-2.6)
   → cp (not mv) staged scripts into place, preserving the original staged path (§2.7)
   → declare params: in train so param changes are tracked (§2.8)
   → mark metrics/reports cache:false — tracked, not pushed as cache objects (§2.9)
   → dvc repro → verify dvc status reports up to date
   → dvc push → operational proof the remote actually works (§2.4)
   → git add -A && git commit → capture the full project state
   → git tag -a v1.0 → name the already-committed, already-pushed state as the release (§2.11)
```

---

## 3. Concepts (reference)

### 3.1 S3-compatible protocol vs. storage backend
DVC's `s3://` remote type speaks a protocol, not a specific vendor.
Any service implementing the S3 API — AWS S3, SeaweedFS, MinIO — can
sit behind it. The `endpointurl` config is what actually pins the
protocol to a specific backend.

### 3.2 Pipeline contract: producer path vs. declared output path
A DVC stage's command can write anywhere on disk; `dvc.yaml`'s `outs:`
is a separate declaration of what DVC will look for afterward. The
two must agree by exact path — DVC has no fuzzy matching between "what
got written" and "what was declared."

### 3.3 `params:` as an explicit dependency declaration
Listing keys under a stage's `params:` tells DVC to treat
`params.yaml` values as dependencies for staleness detection —
distinct from the script simply reading the file at runtime, which
DVC has no visibility into on its own.

### 3.4 `cache: false` on metrics/outputs
Marks a file as DVC-tracked for versioning/metrics purposes without
adding it to the content-addressed cache or default `dvc push`
payload — appropriate for small, human-readable, frequently-diffed
files as opposed to large binary artifacts.

### 3.5 Git tag as a release marker, not an artifact action
A tag only names a commit; it performs no upload, no rerun, and no
staging. Everything the release actually depends on (pushed artifacts,
committed metadata) must already be true before the tag is created.

### 3.6 Pipeline/metric success vs. model quality
`dvc status` and `dvc repro` succeeding verify internal consistency —
that declared dependencies, scripts, and outputs agree with each
other. They say nothing about whether the resulting model is
statistically sound; that requires separate, domain-level judgment
(sample size, class balance, evaluation methodology).

---

## 4. Runbook

### 4.1 Inspect the existing pipeline and remote
```bash
dvc remote list
```
```text
s3      s3://dvc-storage        (default)
```
```bash
dvc config --list
```
```text
remote.s3.url=s3://dvc-storage
remote.s3.endpointurl=http://localhost:8333
core.remote=s3
```
Confirms the remote is SeaweedFS's S3-compatible endpoint, not AWS S3.

### 4.2 Diagnose the broken preprocessing stage
```bash
dvc repro
```
```text
Preprocessed: 20 clean rows
ERROR: failed to reproduce 'preprocess':
output 'data/processed/cleaned.csv' does not exist
```
```bash
cat scripts/preprocess.py
```
```python
df.to_csv("data/processed/clean.csv", index=False)
```
Mismatch: script writes `clean.csv`; `dvc.yaml` declared
`cleaned.csv`. Corrected the `preprocess` output declaration to
`data/processed/clean.csv`, matching what `train.py` already expects.

### 4.3 Stage the training/evaluation scripts without removing the originals
```bash
cp scripts-staging/train.py scripts/train.py
cp scripts-staging/evaluate.py scripts/evaluate.py
```

### 4.4 Declare the full five-stage pipeline
Final `dvc.yaml` stages:

| Stage | Purpose | Key output |
|---|---|---|
| `ingest` | Ingest raw transaction data | raw data available downstream |
| `validate` | Check data validity | `reports/validation.json` |
| `preprocess` | Remove duplicate/missing rows | `data/processed/clean.csv` |
| `train` | Train random forest model | `models/model.pkl`, `data/processed/test_split.csv`, `metrics.json` |
| `evaluate` | Evaluate saved model on test split | `reports/evaluation.json` |

`train` stage params (already present in `params.yaml`):
```yaml
params:
  - n_estimators
  - max_depth
  - test_size
  - random_seed
```
```yaml
test_size: 0.2
random_seed: 42
n_estimators: 100
max_depth: 5
```

Metrics/report outputs marked uncached:
```yaml
metrics:
  - metrics.json:
      cache: false
```
```yaml
outs:
  - reports/evaluation.json:
      cache: false
```

### 4.5 Reproduce the pipeline
```bash
dvc repro
```
```text
Data ingested successfully: 20 rows, 5 columns
Validation: 20 rows, valid=True
Preprocessed: 20 clean rows
Trained: {'accuracy': 1.0, 'f1_score': 1.0}
Evaluation: {'accuracy': 1.0, 'f1_score': 1.0, 'precision': 1.0, 'recall': 1.0, 'test_samples': 4}
```

### 4.6 Verify pipeline consistency
```bash
dvc status
```
```text
Data and pipelines are up to date.
```

### 4.7 Push artifacts to the SeaweedFS remote
```bash
dvc push
```
```text
Collecting |3.00
Pushing
3 files pushed
```

### 4.8 Commit the release
```bash
git add -A
git commit -m "Complete production DVC pipeline with SeaweedFS remote"
```
```text
525e29f Complete production DVC pipeline with SeaweedFS remote
```

### 4.9 Tag the release
```bash
git tag -a v1.0 -m "Production DVC pipeline v1.0"
```
```text
525e29f0fb95365fbd7e262a55c3be71ad4bc96f (HEAD -> main, tag: v1.0)
```

### 4.10 Final verification
```bash
git status --short --branch
git log -1 --oneline
git show v1.0 --stat
dvc status
dvc metrics show
dvc remote list
```

---

## 5. Final state

| Requirement | Result |
|---|---|
| All 5 stages execute and produce declared outputs | ✅ |
| `params:`/`metrics:`/`cache: false` declarations correct | ✅ |
| Staged scripts copied, originals preserved in `scripts-staging/` | ✅ |
| `dvc status` → up to date | ✅ |
| `dvc push` → 3 cache objects pushed to SeaweedFS | ✅ |
| Release committed (`525e29f`) and tagged (`v1.0`) | ✅ |
| Working tree clean | ✅ |

```text
Git                              DVC                          SeaweedFS
 main (tag: v1.0, 525e29f)        .dvc + dvc.lock metadata      s3://dvc-storage
   │                                     │                       (3 cache objects)
   └── dvc.yaml, params.yaml,            └── cache hashes for ───┘
       scripts/, metrics.json,               models/model.pkl, etc.
       reports/*.json
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `output '<path>' does not exist` after a script prints success | Script's actual write path disagrees with `dvc.yaml`'s declared `outs:` | Compare the script's write path against the declaration; align one to the other |
| Lab checker reports a staged file missing after setup | Used `mv` instead of `cp` on a pre-staged file | Re-stage with `cp`, preserving the original path |
| Can't tell which storage service a DVC remote points to | `dvc remote list` only shows the S3-compatible URL, not the backend | `dvc config --list`, check `endpointurl` |
| `dvc push` reports fewer files than expected | Some outputs are declared `cache: false` and intentionally excluded | Confirm expected count against only the cacheable outputs |
| Training stage doesn't rerun after a `params.yaml` edit | The changed param isn't declared under that stage's `params:` | Add the param key to the stage's `params:` list in `dvc.yaml` |
| Perfect metrics on a tiny test split | Test set too small to be statistically meaningful | Treat as a lab-scale sanity check, not production validation |
| Tag doesn't reflect the final pipeline state | Tagged before committing, or committed before pushing | Always order: push → commit → tag |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.12) out loud
      from "diagnose the SeaweedFS remote" to "tag the release."
- [ ] Explain, in one sentence, why `dvc remote list` alone can't
      confirm which storage backend DVC is actually talking to.
- [ ] Explain why the `preprocess` stage failed even though its script
      printed a success message.
- [ ] Explain why `cp` was the correct choice over `mv` when staging
      `train.py` and `evaluate.py`.
- [ ] Explain why `metrics.json` and `models/model.pkl` are declared
      differently (`cache: false` vs. default caching) in `dvc.yaml`.
- [ ] Explain why the release was pushed and committed *before* being
      tagged, not after.
- [ ] From memory, introduce a sixth stage (`report`) that consumes
      `reports/evaluation.json` and produces a Markdown summary,
      declare it correctly in `dvc.yaml`, rerun, push, and tag as
      `v1.1` — without copying any command from this file.

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
