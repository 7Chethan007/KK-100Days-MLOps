# Day 15 — Parameterize a DVC Pipeline

A KodeKloud "100 MLOps" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive the fix
yourself*.

This is the direct sequel to Day 14 (`process_data → split_data →
train` as a DVC pipeline). That lab wired `deps`/`outs` for files; today
adds the one dependency type files alone can't express — a
configuration *value* a script reads, not a file it opens.

---

## 1. Scenario

The `fraud-detection` pipeline already has three stages:
`process_data → split_data → train`. The training script reads
`n_estimators` from `params.yaml`, but `dvc.yaml`'s `train` stage never
declared that value as a dependency — so changing it wouldn't
necessarily trigger a rerun. Task: declare the parameter explicitly in
`dvc.yaml`, without modifying any Python code, and verify that changing
`n_estimators` now correctly triggers *only* the `train` stage to
rerun.

---

## 2. Reasoning model — how to *derive* the fix, not memorize it

### 2.1 Python already knows about the parameter — DVC does not, automatically

```python
RandomForestClassifier(n_estimators=n_estimators)   # train.py already reads params.yaml
```

The training script fully understands and uses `n_estimators` — that
was never the problem. DVC's staleness detection, though, only tracks
what's explicitly declared in `dvc.yaml`'s `deps`/`outs`/`params` —
**it does not read your Python source and infer what configuration
values matter to it.** A script can depend on a value in every
meaningful sense while DVC remains completely unaware, unless that
dependency is declared in DVC's own terms.

### 2.2 Why the task forbids touching `train.py`

The fix the task wants is entirely in `dvc.yaml`, not in the training
code — because the training code was never broken. This is the same
"locate which layer actually owns the problem" discipline running
through this whole series: Python already correctly *uses* the
parameter; DVC just needed to be told the parameter is a *dependency*.
Editing `train.py` would be fixing a layer that was never wrong.

### 2.3 Before the fix: the dependency graph has a gap

```yaml
train:
  cmd: python3 src/models/train.py
  deps:
    - data/processed/train.csv
    - src/models/train.py
  outs:
    - models/model.pkl
```

```text
train.csv ───────┐
                 ├──> train ───> model.pkl
train.py ────────┘

n_estimators  (exists in params.yaml, but NOT wired into this graph at all)
```

DVC's dependency graph (same DAG concept from Day 14) only has edges
for what's declared — `n_estimators` sits outside it entirely until
`params:` is added. Changing it would be invisible to `dvc status`/`dvc
repro`, silently leaving a stale `model.pkl` that doesn't actually
correspond to the current `params.yaml`.

### 2.4 The fix: `params:` as a third kind of stage dependency

```yaml
train:
  cmd: python3 src/models/train.py
  deps:
    - data/processed/train.csv
    - src/models/train.py
  params:
    - n_estimators
  outs:
    - models/model.pkl
```

```text
deps:   → FILE dependencies (data, code — Day 14's two kinds)
params: → VALUE dependencies, read from params.yaml specifically
outs:   → what the stage produces
```

`params:` is DVC's dedicated mechanism for "this stage's correctness
depends on this *specific key* inside `params.yaml`" — a third
dependency type alongside `deps`'s files, specifically because a
config value isn't a file you can hash by path, but DVC still needs a
way to know it matters.

### 2.5 `params.yaml` as the configuration layer, separate from code

```text
params.yaml  → WHAT value to use         (configuration, no logic)
train.py     → HOW to use that value       (logic, reads the config)
dvc.yaml     → WHETHER a value is tracked as a dependency (DVC's metadata)
```

Keeping the actual number (`n_estimators: 100`) outside the Python
script is what lets you change model configuration without touching
code at all — the same "separate data from logic" principle Day 9's
Cookiecutter lab made explicit for template variables, here applied to
ML hyperparameters instead of project-generation inputs.

### 2.6 First `dvc repro` — establishing the baseline `dvc.lock` entry for params

```bash
dvc repro
```
```text
Trained RandomForestClassifier with n_estimators=100
```

After this run, `dvc.lock` records not just file hashes (Day 14) but
also the specific parameter value used:

```yaml
params:
  params.yaml:
    n_estimators: 100
```

This is the baseline `dvc repro` needs in order to later detect a
*change* — without this recorded, DVC has nothing to compare a future
`params.yaml` edit against.

### 2.7 Changing the parameter and re-running — only `train` reruns

```bash
sed -i 's/n_estimators: 100/n_estimators: 200/' params.yaml
dvc repro -v
```
```text
Stage 'process_data' didn't change, skipping
Stage 'split_data' didn't change, skipping
Dependency 'params.yaml' of stage: 'train' changed
stage: 'train' changed.
Running stage 'train'
Trained RandomForestClassifier with n_estimators=200
```

This is the actual payoff of declaring `params:` correctly. `n_estimators`
has no relationship whatsoever to `process_data` or `split_data`'s own
declared deps/outs — changing it cannot possibly make those stages
stale, so DVC correctly skips them and re-runs *only* the one stage
whose declared dependencies actually changed. This is incremental
pipeline execution exactly as introduced conceptually in Day 14
(§2.10's "unchanged stages get skipped"), now demonstrated concretely
with a non-file dependency as the trigger.

### 2.8 Why this specific selectivity matters at real scale

```text
10-stage pipeline, change ONE hyperparameter in ONE training stage
   → without params: declared → DVC has no idea anything changed → stale model silently kept
   → with params: declared     → DVC reruns EXACTLY the affected stage, nothing upstream, nothing unrelated
```

For a pipeline with expensive stages (data processing on large
datasets, long training runs), correctly-scoped `params:` declarations
are what prevent either of two bad outcomes: silently running on stale
config (if undeclared), or needlessly re-running unrelated expensive
stages (if the dependency graph were cruder than it needs to be).

### 2.9 Verifying the change landed correctly — three independent checks

```bash
cat params.yaml                              # the SOURCE value
grep -A3 -B1 "n_estimators" dvc.lock          # the RECORDED value DVC tracked
ls -lh models/model.pkl                        # the ARTIFACT actually regenerated
dvc status                                      # DVC's own "is everything consistent" verdict
```

Four separate facts, each confirming a different part of the claim —
same "verify multiple independent signals, not just one" discipline as
the bidirectional/multi-layer verification running through this entire
series (AWS resource relationships, Day 13's iptables packet counters,
Day 11's content-hash checks).

### 2.10 The compressed reasoning chain

```text
Requirement (train stage reruns when n_estimators changes, train.py untouched)
   → Confirm train.py already reads n_estimators from params.yaml       → code is fine
   → Inspect train stage in dvc.yaml                                      → NO params: declared — the actual gap
   → Add params: [n_estimators] to the train stage ONLY (not process_data/split_data)
   → dvc repro                                                             → baseline run; dvc.lock records n_estimators: 100
   → sed -i change params.yaml to 200
   → dvc repro -v                                                           → process_data/split_data SKIP; train RUNS (params.yaml changed)
   → Verify: cat params.yaml, grep dvc.lock, ls model.pkl, dvc status        → source, recorded, artifact, and consistency all agree
```

---

## 3. Concepts (reference)

### 3.1 `deps` vs. `params` as two distinct dependency types
`deps` declares file-based dependencies, identified by content hash
(Day 11's mechanism). `params` declares specific keys inside a params
file (conventionally `params.yaml`) as dependencies, tracked by their
actual value rather than a file-level hash — necessary because DVC
needs to know *which key inside the file* a stage cares about, not just
that the file as a whole changed.

### 3.2 Why DVC can't infer code-level config usage automatically
DVC's staleness detection operates purely on the declared graph in
`dvc.yaml` — it has no static-analysis step that reads Python source to
discover which config values a script actually consults. Any value a
script depends on for correctness must be explicitly declared via
`params:` for DVC to track it at all.

### 3.3 `params.yaml` as a configuration layer separate from code
Keeping tunable values (hyperparameters, thresholds, paths) outside
source code is the same "separation of data from logic" principle
behind Day 9's Cookiecutter template variables — it's what lets a value
change without a code change, and (once `params:` is declared) lets DVC
track that change as a first-class pipeline event.

### 3.4 Incremental execution scoped correctly (recap + extension of Day 14)
A stage only reruns if something in *its own* declared `deps`/`params`
changed — unrelated stages elsewhere in the DAG are unaffected,
regardless of how the dependency graph as a whole looks. Declaring
`params:` only on the stage that actually uses that parameter is what
keeps this scoping precise.

### 3.5 `dvc.lock`'s `params:` section
Alongside recorded file hashes, `dvc.lock` also records the exact
parameter value(s) used in the last successful run of each stage that
declares `params:` — this is the baseline against which a future
`params.yaml` edit is compared to decide staleness.

---

## 4. Runbook

### 4.1 Inspect the current pipeline and parameter file
```bash
cd /root/code/fraud-detection
cat params.yaml
```
```yaml
n_estimators: 100
```
```bash
cat dvc.yaml
```
Confirm the `train` stage currently has `deps`/`outs` but no `params:`.

### 4.2 Add the parameter dependency to the `train` stage only
```yaml
train:
  cmd: python3 src/models/train.py
  deps:
    - data/processed/train.csv
    - src/models/train.py
  params:
    - n_estimators
  outs:
    - models/model.pkl
```
(`process_data` and `split_data` are left untouched — they have no
relationship to `n_estimators`.)

### 4.3 Run the baseline reproduction
```bash
dvc repro
```
```text
Trained RandomForestClassifier with n_estimators=100
```

### 4.4 Confirm the baseline parameter value is recorded
```bash
grep -A3 -B1 "n_estimators" dvc.lock
```
```yaml
params.yaml:
  n_estimators: 100
```

### 4.5 Change the parameter
```bash
sed -i 's/n_estimators: 100/n_estimators: 200/' params.yaml
cat params.yaml
```
```yaml
n_estimators: 200
```

### 4.6 Reproduce again and observe selective re-execution
```bash
dvc repro -v
```
```text
Stage 'process_data' didn't change, skipping
Stage 'split_data' didn't change, skipping
Dependency 'params.yaml' of stage: 'train' changed
stage: 'train' changed.
Running stage 'train'
Trained RandomForestClassifier with n_estimators=200
```

### 4.7 Verify every independent signal
```bash
cat params.yaml
```
```yaml
n_estimators: 200
```
```bash
grep -A3 -B1 "n_estimators" dvc.lock
```
```yaml
params.yaml:
  n_estimators: 200
```
```bash
ls -lh models/model.pkl
```
```text
-rw-r--r-- 1 root root 136K ... models/model.pkl
```
```bash
dvc status
```
```text
Data and pipelines are up to date.
```

### 4.8 Submit
With `params:` declared only on `train`, `dvc repro -v` showing exactly
`train` rerunning after the `params.yaml` edit, and all four
verification signals agreeing, click **Check** in the KodeKloud lab UI.

---

## 5. Final state

| Requirement | Result |
|---|---|
| `train` stage declares `params: [n_estimators]` | ✅ |
| `process_data`/`split_data` left unchanged (no spurious `params:`) | ✅ |
| `train.py` left unmodified | ✅ |
| Changing `n_estimators` triggers rerun of `train` ONLY | ✅ |
| `dvc.lock` records the current parameter value | ✅ |
| `model.pkl` regenerated | ✅ |
| `dvc status` reports up to date after the change | ✅ |

```text
params.yaml: n_estimators = 200
        │
        │ declared as a PARAMS dependency (not a file dep)
        ▼
dvc.yaml: train.params = [n_estimators]
        │
        ▼
dvc repro -v
        │
        ├── process_data → SKIP (no relationship to n_estimators)
        ├── split_data   → SKIP (no relationship to n_estimators)
        └── train         → RUN  (params.yaml changed)
                 │
                 ▼
         model.pkl regenerated
                 │
                 ▼
         dvc.lock: params.yaml.n_estimators = 200
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Changed `n_estimators` but `dvc status`/`dvc repro` don't notice | `params:` was never declared on the `train` stage | Add `params: [n_estimators]` to exactly the stage that uses it |
| `process_data` or `split_data` unexpectedly reruns after a parameter change | `params:` mistakenly added to a stage that doesn't actually use that value | Only declare `params:` on the stage(s) whose script genuinely reads that key |
| Edited `train.py` to "fix" the staleness problem | Misdiagnosed the problem as a code issue instead of a missing DVC dependency declaration | Revert the code edit; the fix belongs entirely in `dvc.yaml` (§2.2) |
| `dvc.lock` shows the old parameter value after changing `params.yaml` and running `dvc repro` | `dvc repro` wasn't actually re-run after the edit, or the edit targeted the wrong key/file | Re-run `dvc repro -v` and read its output to confirm it detected the change |
| Unsure whether the rerun actually used the new parameter | Only checked `dvc status`, not the model artifact or script's own printed output | Check the script's own log line (`Trained ... with n_estimators=...`) and/or `dvc.lock`'s recorded value directly |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.10) out loud
      from "changing n_estimators doesn't trigger a rerun" to "verified
      train reruns selectively, all four signals agree."
- [ ] Explain, in one sentence, why DVC can't automatically know that
      `train.py` depends on `n_estimators` just by reading the script.
- [ ] Explain the difference between `deps` and `params` as dependency
      types — why can't a parameter just be listed under `deps`?
- [ ] Explain why `process_data` and `split_data` correctly don't rerun
      when `n_estimators` changes, using the DAG concept from Day 14.
- [ ] Explain why the fix belongs entirely in `dvc.yaml` and not in
      `train.py`, given that `train.py` already worked correctly.
- [ ] Add a second parameter (e.g. `max_depth`) to `params.yaml`,
      declare it on the `train` stage from memory, and verify changing
      it alone also triggers exactly one stage to rerun.

---

## 8. Document template for future lab days

Use this skeleton for `day-N/README.md`:

```markdown
# Day N — <task title>

## 1. Scenario
<what was asked, what was broken/required, constraints>

## 2. Reasoning model — how to derive the fix/commands
<walk the requirement down step by step, in the order you'd actually
discover it: confirm which layer (code vs. metadata/config) actually
owns the gap → identify the missing dependency declaration → add it
scoped to exactly the stage that needs it → run a baseline → trigger
the change → observe selective re-execution → verify multiple
independent signals agree. End with a compressed arrow-chain version.>

## 3. Concepts (reference)
<one subsection per concept the task exercises — explain WHY, not just WHAT>

## 4. Runbook
<copy-pasteable commands in the order run, with real intermediate
output/results inline where they mattered to the diagnosis>

## 5. Final state
<table + diagram of what the pipeline/repository looks like after
completion>

## 6. Troubleshooting
<symptom / cause / fix table>

## 7. Do it yourself (checklist)
<"walk the reasoning chain out loud" + comprehension questions + "extend
it yourself" prompt>
```
