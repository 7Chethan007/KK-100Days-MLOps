# Day 17 — Run and Compare DVC Experiments

A KodeKloud "100 MLOps" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how experiments relate to
the pipeline/params/metrics machinery from Days 14–16*.

This is the direct sequel to Day 16 (`metrics:` reclassification). Day
16 taught DVC to *understand* `metrics.json` as queryable results;
today uses that understanding to systematically compare multiple
hyperparameter choices and promote the best one.

---

## 1. Scenario

Baseline configuration in `params.yaml`:

```yaml
n_estimators: 100
max_depth: 4
```
```text
accuracy = 0.71
f1_score = 0.5735
```

Task: test `max_depth` at `2`, `6`, and `12` as DVC experiments,
compare their `f1_score`, and promote the winning configuration into
the workspace — without manually editing `params.yaml` and re-running
by hand for each trial.

| Experiment | `max_depth` | accuracy | f1_score |
|---|---:|---:|---:|
| baseline (`main`) | 4 | 0.710 | 0.5735 |
| `owned-tirl` | 2 | 0.600 | 0.2453 |
| `rapid-keep` | 6 | 0.745 | 0.6483 |
| **`wired-nuke`** | **12** | **0.750** | **0.6795** |

---

## 2. Reasoning model — how to *derive* the fix, not memorize it

### 2.1 The problem `dvc exp run` solves

```text
Naive workflow:
edit params.yaml → run training → record result by hand → restore params
→ edit again → run again → ... (repeat per trial, manually)
```

Manually cycling through configurations and recording results
somewhere is exactly the kind of bookkeeping DVC is built to replace.
`dvc exp run -S <param>=<value>` runs the pipeline with a *temporary*
parameter override — it never requires hand-editing `params.yaml` for
each trial, and it automatically records the resulting metrics against
the exact parameter value that produced them.

### 2.2 Why only `train` reruns per experiment — direct reuse of Day 14/15 mechanics

```text
process_data   → no relationship to max_depth → SKIP
split_data     → no relationship to max_depth → SKIP
train          → params: [n_estimators, max_depth] → RERUN
```

This is not new behavior introduced by experiments — it's the exact
same DAG/`params:`-dependency mechanism from Day 14 (stage chaining)
and Day 15 (`params:` declarations), now triggered by `-S` overrides
instead of a direct `params.yaml` edit. Recognizing this means you
don't need a separate mental model for "how do experiments decide what
reruns" — it's identical to how `dvc repro` always decided it.

### 2.3 An experiment is an isolated, comparable trial — not an automatic commit

```text
dvc exp run -S max_depth=6
        │
        ▼
An isolated variation of the pipeline's current state, with its own
parameters/metrics/model — NOT yet part of your Git history, and NOT
yet the workspace's "real" state.
```

Running an experiment never touches `main`'s Git history and never
permanently overwrites your workspace's tracked files in a way Git
would register as a change — it's deliberately disposable and
comparable, which is what makes running three (or thirty) of them
*safe* to do freely.

### 2.4 Experiment name vs. experiment ID — two different identifiers for the same trial

```text
wired-nuke   → a human-readable, auto-generated NAME
0ee3809      → the actual experiment ID (short hash)
```

`dvc exp show` displays both; `dvc exp apply` can take either, but the
ID is the more precise, stable identifier — worth not confusing the
two when scripting or referring back to a specific trial later.

### 2.5 `dvc exp show` vs. `dvc metrics show` vs. `dvc metrics diff` — three different scopes

```text
dvc metrics show   → "what are CURRENT WORKSPACE metrics, right now?"
dvc exp show        → "what are ALL EXPERIMENTS' parameters+metrics, compared?"
dvc metrics diff     → "how did metrics change between two SPECIFIC revisions?"
```

Each answers a genuinely different question — `dvc exp show` is the
one actually needed here, since the task is comparing *several*
trials against each other, not just reading one current state
(`metrics show`) or diffing exactly two points (`metrics diff`).

### 2.6 Why deeper isn't automatically better — the bias-variance framing behind these specific numbers

```text
max_depth=2   → F1 0.2453   (too shallow — underfitting: the tree can't
                              capture enough pattern to separate classes)
max_depth=6   → F1 0.6483   (meaningfully better — more capacity)
max_depth=12  → F1 0.6795   (best observed — still improving here, but
                              this is exactly the region where further
                              increases could start overfitting instead)
```

The three chosen depths (2, 6, 12) aren't arbitrary — they're probing
across the underfitting-to-adequate-capacity range specifically. The
winner being the *largest* value tested doesn't mean "always maximize
`max_depth`" — it means this particular range hadn't yet crossed into
the overfitting side for this dataset; a real investigation would keep
testing beyond 12 to find where F1 eventually turns back down.

### 2.7 Why F1, not accuracy, is the deciding metric for fraud detection

```text
accuracy  → can look deceptively high on an imbalanced dataset
             (predicting "not fraud" for everything scores ~99% if
              fraud is rare)
f1_score   → balances precision and recall, far more informative when
              the positive class (fraud) is the minority
```

This is why the task's comparison criterion is explicitly `f1_score`,
not `accuracy` — on this exact kind of problem (rare positive class),
accuracy alone can hide a model that's actually useless at the thing
that matters.

### 2.8 `dvc exp apply` — the explicit, deliberate promotion step

```bash
dvc exp apply 0ee3809
```
```text
Changes for experiment '0ee3809' have been applied to your current workspace.
```

Nothing is promoted automatically, ever — `dvc exp apply` is the one
explicit action that takes a specific experiment's parameters/
metrics/model and makes them the actual workspace state. Even after
this, no Git commit has happened yet (§2.9) — `apply` only affects the
working tree.

### 2.9 Experiment promotion and Git commit are still two separate decisions

```text
dvc exp run (x N)   → disposable trials, compared freely
       │
       ▼
dvc exp apply <id>   → chosen trial becomes the WORKSPACE state
       │
       ▼
git commit            → a SEPARATE, deliberate decision to record
                          this promoted state in history
```

This two-step separation is what lets you run many throwaway
experiments without polluting Git history with every failed attempt —
only the one you deliberately choose to commit ever becomes part of
the permanent record.

### 2.10 The compressed reasoning chain

```text
Requirement (test max_depth 2/6/12, compare F1, promote the winner)
   → Confirm clean baseline: dvc status, dvc metrics show, git status --short
   → dvc exp run -S max_depth=2    → owned-tirl: F1 0.2453 (worse — underfitting)
   → dvc exp run -S max_depth=6    → rapid-keep: F1 0.6483 (better)
   → dvc exp run -S max_depth=12   → wired-nuke: F1 0.6795 (best)
   → dvc exp show                    → compare all trials in one table
   → Identify the winner by F1 (not accuracy) → wired-nuke, ID 0ee3809
   → dvc exp apply 0ee3809             → promote into the WORKSPACE (not yet committed)
   → Verify: cat params.yaml (max_depth: 12), dvc metrics show (0.75 / 0.6795)
   → (a real workflow would then: git commit — a separate, deliberate step)
```

---

## 3. Concepts (reference)

### 3.1 Hyperparameter vs. model parameter
A model parameter (split thresholds, learned weights) is determined
*by* the training algorithm. A hyperparameter (`max_depth`,
`n_estimators`, learning rate) is chosen *before/around* training and
controls how that algorithm behaves — this lab is entirely about
searching over hyperparameter choices, never touching the model's
internally-learned parameters directly.

### 3.2 `dvc exp run -S <param>=<value>`
Runs the pipeline with a temporary override of one or more declared
`params:` values, without permanently editing `params.yaml`. Only
stages whose declared `params:`/`deps` are affected by the override
actually rerun (§2.2).

### 3.3 Experiment vs. workspace vs. Git commit — three distinct states
Experiment = an isolated, comparable trial. Workspace = the files
currently on disk, as seen by `dvc status`/`git status`. Git commit =
the permanently recorded history. `dvc exp apply` moves state from
experiment → workspace; `git commit` is the separate step that moves
workspace → history.

### 3.4 `dvc exp show`
Displays every known experiment (plus `workspace` and the current Git
revision) in one comparison table, with their parameters and metrics
side by side — the primary tool for deciding which trial actually won.

### 3.5 Precision, recall, and F1 (brief)
```text
Precision = TP / (TP + FP)
Recall    = TP / (TP + FN)
F1        = 2 × Precision × Recall / (Precision + Recall)
```
F1 is the harmonic mean of precision and recall — it penalizes models
that sacrifice one heavily for the other, making it a better single
metric than accuracy alone for imbalanced classification tasks like
fraud detection.

### 3.6 Held-out test set evaluation
The model is evaluated on data it never saw during training
(`test.csv`, produced by `split_data` back in Day 14's pipeline) — this
is what makes the F1 comparison across experiments a meaningful
estimate of real-world generalization, rather than a number that could
be trivially inflated by memorizing the training data.

---

## 4. Runbook

### 4.1 Inspect the project and confirm a clean baseline
```bash
cd /root/code/fraud-detection
cat params.yaml
```
```yaml
n_estimators: 100
max_depth: 4
```
```bash
dvc status
```
```text
Data and pipelines are up to date.
```
```bash
dvc metrics show
```
```text
Path          accuracy    f1_score
metrics.json  0.71        0.5735
```
```bash
git status --short
```
(no output — clean working tree before starting)

### 4.2 Run experiment 1 — max_depth=2
```bash
dvc exp run -S max_depth=2
```
Result (`owned-tirl`):
```text
max_depth    = 2
n_estimators = 100
accuracy     = 0.60
f1_score     = 0.2453
```

### 4.3 Run experiment 2 — max_depth=6
```bash
dvc exp run -S max_depth=6
```
Result (`rapid-keep`):
```text
max_depth    = 6
n_estimators = 100
accuracy     = 0.745
f1_score     = 0.6483
```

### 4.4 Run experiment 3 — max_depth=12
```bash
dvc exp run -S max_depth=12
```
Result (`wired-nuke`):
```text
max_depth    = 12
n_estimators = 100
accuracy     = 0.75
f1_score     = 0.6795
```

### 4.5 Compare all experiments
```bash
dvc exp show
```
```text
Experiment     accuracy   f1_score   n_estimators   max_depth
workspace      0.75       0.6795     100            12
main           0.71       0.5735     100            4
wired-nuke     0.75       0.6795     100            12
rapid-keep     0.745      0.6483     100            6
owned-tirl     0.6        0.2453     100            2
```

### 4.6 Identify and promote the winner
Winner by `f1_score`: `wired-nuke` (ID `0ee3809`).
```bash
dvc exp apply 0ee3809
```
```text
Changes for experiment '0ee3809' have been applied to your current workspace.
```

### 4.7 Verify the promoted state
```bash
cat params.yaml
```
```yaml
n_estimators: 100
max_depth: 12
```
```bash
dvc metrics show
```
```text
Path          accuracy    f1_score
metrics.json  0.75        0.6795
```

### 4.8 Submit
With the three experiments run, compared via `dvc exp show`, the
highest-`f1_score` experiment (`wired-nuke`) applied to the workspace,
and `params.yaml`/`metrics.json` confirmed to reflect `max_depth=12`,
click **Check** in the KodeKloud lab UI.

---

## 5. Final state

| Requirement | Result |
|---|---|
| Baseline recorded before experimenting | ✅ `max_depth=4`, F1 0.5735 |
| `max_depth=2` tested as an experiment | ✅ `owned-tirl`, F1 0.2453 |
| `max_depth=6` tested as an experiment | ✅ `rapid-keep`, F1 0.6483 |
| `max_depth=12` tested as an experiment | ✅ `wired-nuke`, F1 0.6795 (winner) |
| Winner identified by `f1_score`, not accuracy | ✅ |
| Winning experiment applied to workspace | ✅ `dvc exp apply 0ee3809` |
| Workspace `params.yaml`/`metrics.json` reflect the winner | ✅ |

```text
Baseline (max_depth=4, F1=0.5735)
        │
        ├── dvc exp run -S max_depth=2   → owned-tirl  (F1 0.2453)
        ├── dvc exp run -S max_depth=6   → rapid-keep  (F1 0.6483)
        └── dvc exp run -S max_depth=12  → wired-nuke  (F1 0.6795) ← winner
                        │
                        │ dvc exp apply 0ee3809
                        ▼
                   Workspace: max_depth=12, F1=0.6795
                        │
                        ▼
              (git commit — separate, deliberate step)
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Confused an experiment's name with its ID when running `dvc exp apply` | Both appear in `dvc exp show`, easy to conflate | Confirm you're using the exact identifier `dvc exp show` lists under the ID column, not just the name, if the apply command is picky |
| Assumed the workspace already reflected the winning experiment without applying it | `dvc exp run` never automatically promotes anything | Always run `dvc exp apply <id>` explicitly, then re-verify `params.yaml`/`metrics.json` |
| Compared experiments by `accuracy` instead of `f1_score` | Accuracy can look deceptively high/stable on an imbalanced dataset | Re-check `dvc exp show`'s `f1_score` column specifically for this task |
| Assumed a higher `max_depth` always wins | Only three points were tested; this dataset's F1 was still improving at 12, but that doesn't generalize to "always maximize depth" | Test further values (e.g. 16, 20) if a genuinely optimal depth is needed — don't extrapolate from three points |
| `process_data`/`split_data` unexpectedly reran during an experiment | A dependency those stages DO declare (data file, their own script) changed, unrelated to `max_depth` | Re-check what actually changed; `max_depth` alone should never trigger those stages |
| Ran `dvc exp apply` but `git status` shows nothing to commit | Applying an experiment changes tracked files' *content*, but if the applied experiment's state happens to match what's already committed, there's genuinely nothing new to commit | Confirm `params.yaml`/`metrics.json` content directly rather than relying on `git status` alone |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.10) out loud
      from "test max_depth at 2/6/12" to "winning experiment applied,
      verified in the workspace."
- [ ] Explain, in one sentence, why only the `train` stage reran for
      each experiment, referencing the `params:` mechanism from Day 15.
- [ ] Explain the difference between an experiment's name and its ID.
- [ ] Explain why `dvc exp run` never automatically becomes a Git
      commit, and what two separate steps actually get you there.
- [ ] Explain why F1, not accuracy, was the correct metric to compare
      across these experiments.
- [ ] Explain the difference between `dvc exp show`, `dvc metrics
      show`, and `dvc metrics diff` — which one would you reach for to
      compare five experiments at once?
- [ ] Run two more experiments from memory (e.g. `max_depth=16` and
      `max_depth=20`), compare all five via `dvc exp show`, and
      determine whether F1 is still improving or has started to
      decline — then explain what that result would mean.

---

## 8. Document template for future lab days

Use this skeleton for `day-N/README.md`:

```markdown
# Day N — <task title>

## 1. Scenario
<what was asked, what was broken/required, constraints>

## 2. Reasoning model — how to derive the fix/commands
<walk the requirement down step by step, in the order you'd actually
discover it: establish a clean baseline → run each trial/variation as
an isolated, comparable unit (reusing the same dependency/staleness
mechanics already established in prior labs, not a new mechanism) →
compare ALL trials in one view → choose a winner by the metric that
actually matters for the problem → explicitly promote the winner,
recognizing promotion and permanent commit are two separate decisions.
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
