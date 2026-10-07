# Day 16 — DVC Metrics

A KodeKloud "100 MLOps" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how `metrics:` differs from
`outs:` at a conceptual level, not just a syntax level*.

This continues the DVC arc from Days 10–15 (init → add → push/pull →
pipelines → params). Having wired `n_estimators` as a tracked input in
Day 15, today's lab wires the pipeline's measurable *output* —
accuracy/F1 — as a first-class, queryable DVC concept instead of just
another file sitting in `outs:`.

---

## 1. Scenario

The `fraud-detection` pipeline (`process_data → split_data → train`)
produces both `models/model.pkl` and `metrics.json` from the `train`
stage. `metrics.json` is currently declared as a plain `outs:` entry.
Task: reconfigure it as a DVC **metric** (`cache: false`), without
touching any Python code, then use `dvc metrics show` to display the
model's evaluation results.

```text
train
  ├── model.pkl       → outs:     (a normal artifact)
  └── metrics.json     → metrics:  (a QUERYABLE result, not just a file)
```

---

## 2. Reasoning model — how to *derive* the fix, not memorize it

### 2.1 Every file in a DVC stage's output is currently treated identically — that's the actual gap

```yaml
outs:
  - models/model.pkl
  - metrics.json       # ← same treatment as model.pkl: cached, tracked, opaque
```

DVC doesn't inherently know `metrics.json`'s *content* is different
from `model.pkl`'s — both are just tracked output files under `outs:`.
The gap isn't that DVC can't see `metrics.json` at all (it can, and
does track it) — it's that DVC has no special understanding of it as
*measurable results you'd want to query, compare, or diff across runs*.

### 2.2 `metrics:` as a declaration of *meaning*, not just a different file list

```yaml
outs:
  - models/model.pkl
metrics:
  - metrics.json:
      cache: false
```

Moving `metrics.json` from `outs:` to `metrics:` tells DVC: *"this file
contains structured, numeric results I want to query and compare, not
just an opaque artifact."* This unlocks an entire category of commands
(`dvc metrics show`, `dvc metrics diff`) that simply don't apply to
`model.pkl` — DVC has no idea what's inside a `.pkl` file, but it can
parse a metrics JSON/YAML file's keys and values directly.

### 2.3 Why `cache: false` specifically

```text
cache: true (default)   → DVC caches this file like any other tracked output,
                            treating it as potentially large, content-hashed data
cache: false              → stays a normal, small, Git-friendly file —
                            not routed through DVC's cache/remote mechanism at all
```

Metrics files are typically tiny (a few numbers in JSON) and you
*want* them versioned directly in Git history, diffable commit-to-
commit the normal Git way — routing something this small through DVC's
content-addressable cache (designed for large datasets/models) would
be solving a problem metrics files don't have. `cache: false` keeps
`metrics.json` behaving like an ordinary small file DVC still knows
how to *interpret specially* (§2.2), just without the caching
machinery built for large artifacts.

### 2.4 No Python code changes — same "fix belongs in the metadata layer" lesson as Day 15

The training script already writes `metrics.json` with the correct
content (`accuracy`, `f1_score`) — exactly like Day 15's `train.py`
already correctly read `n_estimators`. The gap, both times, was purely
in how `dvc.yaml` *described* an existing, already-correct piece of
the pipeline. Editing Python here would be fixing a layer that was
never broken.

### 2.5 `dvc repro` after the config change — same mechanics as every prior pipeline lab

```bash
dvc repro
```
```text
Trained model: accuracy=1.0, f1_score=1.0
```

Reclassifying `metrics.json` from `outs:` to `metrics:` is itself a
`dvc.yaml` change, which `dvc repro` detects the same way it detects
any other stage definition change (Day 14's dependency-graph
mechanics) — `train` reruns, regenerating both `model.pkl` and
`metrics.json` under the new classification.

### 2.6 `dvc metrics show` — the actual payoff

```bash
dvc metrics show
```
```text
Path          accuracy    f1_score
metrics.json  1.0         1.0
```

This is the concrete proof the reclassification worked — DVC parsed
`metrics.json`'s contents and displayed its keys/values in a table,
something it would never do for an arbitrary `outs:` file like
`model.pkl`. Getting this output is stronger verification than just
confirming the file exists; it confirms DVC specifically recognizes it
as a metric.

### 2.7 The broader payoff this single lab sets up: `dvc metrics diff`

```text
                old       new       change
accuracy        0.91      0.94      +0.03
f1_score         0.88      0.92      +0.04
```

Once a file is correctly declared as a `metrics:` entry, DVC can
compare its values *across Git revisions* — this is the actual
reason the distinction matters in a real MLOps workflow: "did my last
change to the code/data/params actually improve the model" becomes a
single command (`dvc metrics diff`) instead of manually diffing JSON
files by hand across commits. Today's lab is the prerequisite
declaration that makes that comparison possible at all.

### 2.8 `dvc.yaml` vs. `dvc.lock`, reapplied to metrics (recap from Day 14)

```text
dvc.yaml  → declares metrics.json AS a metric (the intent)
dvc.lock  → records the actual metrics.json hash/state from the last successful run
```

Same intent-vs-recorded-state split as every other DVC artifact type
covered so far (files in Day 11, pipeline stages in Day 14, params in
Day 15) — metrics are not a special exception to this pattern, they're
the same mechanism applied to one more category of pipeline output.

### 2.9 The compressed reasoning chain

```text
Requirement (metrics.json treated as a DVC metric, dvc metrics show works)
   → Inspect dvc.yaml                              → metrics.json under plain outs: (the gap)
   → Confirm train.py already writes correct metrics.json content → code is fine
   → Move metrics.json: outs: → metrics: [metrics.json: {cache: false}]
   → dvc repro                                        → train reruns under the new stage definition
   → ls -lh metrics.json models/model.pkl               → confirm both artifacts regenerated
   → dvc metrics show                                    → DVC parses + displays accuracy/f1_score — proof of reclassification
   → (conceptually) dvc metrics diff across commits        → the actual downstream payoff this unlocks
```

---

## 3. Concepts (reference)

### 3.1 `outs:` vs. `metrics:` — different DVC semantics for different kinds of output
`outs:` declares an opaque tracked artifact (DVC caches/hashes it, has
no notion of its internal structure). `metrics:` declares a file DVC
can parse (JSON/YAML with numeric values) and treat as queryable,
comparable experiment results — unlocking `dvc metrics show`/`diff`
specifically.

### 3.2 `cache: false`
Opts a declared output out of DVC's content-addressable caching
mechanism — appropriate for small, Git-friendly files (like metrics)
that don't need the large-artifact handling DVC's cache is built for.

### 3.3 Why metrics files are typically small and Git-tracked directly
A metrics JSON file is usually a handful of key/value pairs — cheap to
diff directly in Git history, unlike a multi-GB dataset or a large
serialized model. `cache: false` reflects this difference explicitly
in the pipeline definition.

### 3.4 `dvc metrics show` / `dvc metrics diff`
`show` displays current metric values across all declared `metrics:`
files. `diff` compares metric values between two Git revisions (e.g.
the working tree vs. the last commit, or two arbitrary commits) —
the core mechanism for answering "did this change improve the model"
without manual JSON comparison.

### 3.5 Metrics as the bridge between Git history and ML experiment tracking
Because metrics are small, Git-diffable files *and* DVC understands
their structure, every Git commit that changes code/data/params and
reruns the pipeline leaves behind a comparable metrics snapshot —
turning ordinary Git history into a lightweight experiment log.

### 3.6 The broader DVC command surface (context beyond this lab)
- `dvc dag` — see the pipeline graph (Day 14).
- `dvc repro` / `dvc status` — run / check staleness (Day 14).
- `dvc add` / `push` / `pull` / `checkout` — individual artifact
  tracking and remote sync (Days 11–13).
- `dvc params diff` — the `params.yaml`-equivalent of `metrics diff`,
  comparing configuration rather than results (Day 15's natural
  counterpart).
- `dvc gc` — garbage-collect unreferenced cached data; use cautiously,
  especially on shared repositories, since it can remove data DVC no
  longer considers referenced.

---

## 4. Runbook

### 4.1 Navigate to the project and inspect the current pipeline
```bash
cd /root/code/fraud-detection
cat dvc.yaml
```
```yaml
train:
  cmd: python3 src/models/train.py
  deps:
    - data/processed/train.csv
    - src/models/train.py
  outs:
    - models/model.pkl
    - metrics.json
```
`metrics.json` is currently a plain `outs:` entry — the gap.

### 4.2 Reclassify `metrics.json` as a metric
```yaml
train:
  cmd: python3 src/models/train.py
  deps:
    - data/processed/train.csv
    - src/models/train.py
  outs:
    - models/model.pkl
  metrics:
    - metrics.json:
        cache: false
```
No Python files modified.

### 4.3 Reproduce the pipeline
```bash
dvc repro
```
```text
Running stage 'train':
> python3 src/models/train.py
Trained model: accuracy=1.0, f1_score=1.0
```

### 4.4 Verify both generated files
```bash
ls -lh metrics.json models/model.pkl
```
```text
-rw-r--r-- 1 root root  68K ... models/model.pkl
-rw-r--r-- 1 root root   45 ... metrics.json
```

### 4.5 Display the metrics via DVC
```bash
dvc metrics show
```
```text
Path          accuracy    f1_score
metrics.json  1.0         1.0
```
This confirms DVC specifically recognizes `metrics.json` as a metric,
not just a tracked file.

### 4.6 Confirm the pipeline is otherwise consistent
```bash
dvc status
```
```text
Data and pipelines are up to date.
```

### 4.7 Submit
With `metrics.json` declared under `metrics:` with `cache: false`,
`dvc repro` having regenerated both outputs, and `dvc metrics show`
correctly displaying `accuracy`/`f1_score`, click **Check** in the
KodeKloud lab UI.

---

## 5. Final state

| Requirement | Result |
|---|---|
| `metrics.json` declared under `metrics:`, not `outs:` | ✅ |
| `cache: false` set | ✅ |
| `train.py` unmodified | ✅ |
| `dvc repro` regenerates `model.pkl` and `metrics.json` | ✅ |
| `dvc metrics show` displays `accuracy`/`f1_score` | ✅ |
| `dvc status` reports up to date | ✅ |

```text
dvc.yaml: train stage
   ├── outs:    models/model.pkl        (opaque artifact, cached)
   └── metrics: metrics.json            (parsed, queryable, cache: false)
                   │
                   │ dvc repro
                   ▼
          accuracy=1.0, f1_score=1.0
                   │
                   │ dvc metrics show
                   ▼
   Path          accuracy    f1_score
   metrics.json  1.0         1.0
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `dvc metrics show` returns nothing / doesn't recognize `metrics.json` | Still declared under `outs:` instead of `metrics:` | Move the entry to a `metrics:` block in the relevant stage |
| `dvc metrics show` errors trying to parse the file | `metrics.json`'s content isn't valid JSON/YAML, or isn't a flat key-value structure DVC can read | Confirm the training script's output format matches what `dvc metrics show` expects |
| Metrics file ends up cached/hashed like a large artifact | `cache: false` omitted | Add `cache: false` under the metric's entry |
| Edited `train.py` trying to "fix" the metrics recognition | Misdiagnosed the gap as a code issue instead of a `dvc.yaml` declaration issue | Revert; the fix belongs entirely in `dvc.yaml` (§2.4) |
| `dvc repro` doesn't rerun `train` after the `dvc.yaml` edit | Unlikely but possible if DVC doesn't detect the stage definition change | Confirm the YAML is valid and the stage name/structure wasn't accidentally altered beyond the outs/metrics move |
| Confused about why `model.pkl` doesn't show up in `dvc metrics show` | `model.pkl` is still (correctly) an `outs:` artifact — not everything a stage produces should be a metric | Only genuinely measurable, structured result files belong under `metrics:` |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.9) out loud
      from "metrics.json is a plain output" to "`dvc metrics show`
      displays accuracy and f1_score."
- [ ] Explain, in one sentence, the difference between what `outs:`
      and `metrics:` each tell DVC about a file.
- [ ] Explain why `cache: false` is appropriate for `metrics.json` but
      would be inappropriate for `models/model.pkl`.
- [ ] Explain why the fix for this lab lives entirely in `dvc.yaml`,
      given that `train.py` already produced correct metric values.
- [ ] Explain what `dvc metrics diff` would let you do that `dvc
      metrics show` alone cannot.
- [ ] Add a second metric key (e.g. `precision`) to the training
      script's output from memory, confirm it appears in `dvc metrics
      show` without any further `dvc.yaml` changes — why doesn't the
      YAML need to list individual metric keys?

---

## 8. Document template for future lab days

Use this skeleton for `day-N/README.md`:

```markdown
# Day N — <task title>

## 1. Scenario
<what was asked, what was broken/required, constraints>

## 2. Reasoning model — how to derive the fix/commands
<walk the requirement down step by step, in the order you'd actually
discover it: confirm which layer (code vs. pipeline metadata) owns the
gap → identify the specific declaration/classification missing →
change ONLY the metadata layer → re-run → verify via the tool's
OWN specialized command for that category, not just file existence.
End with a compressed arrow-chain version.>

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
