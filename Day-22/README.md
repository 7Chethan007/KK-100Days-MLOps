# Day 22 — Create and Organize MLflow Experiments

A KodeKloud "100 MLOps" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each action exists and *how you'd derive it yourself*.

Day 21 logged one run into the `Default` experiment — the only
experiment that existed at the time. This lab is one level up the
hierarchy: creating and organizing the *experiments themselves*,
with metadata (descriptions, tags) scoped to the experiment as a
whole rather than to any individual run inside it.

---

## 1. Scenario

Register two new experiments on the existing MLflow Tracking Server
via the UI, each tagged with its owning team, while leaving the
existing `Default` and `legacy-models` experiments untouched.

| Experiment | Description | Tag |
|---|---|---|
| `fraud-detection` | "Machine learning project for detecting fraudulent transactions." | `team=ml-platform` |
| `churn-prediction` | not required | `team=analytics` |

```text
MLflow Tracking Server
 ├── Default           (existing — unchanged)
 ├── legacy-models      (existing — unchanged)
 ├── fraud-detection     (new) — description + team=ml-platform
 └── churn-prediction    (new) — team=analytics
```

---

## 2. Reasoning model — how to *derive* the steps, not memorize them

### 2.1 An experiment is a grouping boundary one level above a run

```text
Experiment: fraud-detection
  ├── run: serious-goose-144  (Day 21's run — if it had been logged HERE instead of Default)
  ├── run: <future run 2>
  └── run: <future run 3>
```

Day 21's run landed in `Default` purely because no other experiment
existed at the time and none was explicitly set. Creating
`fraud-detection` and `churn-prediction` as separate experiments means
future training runs for those specific projects can be logged into
the *right* grouping from the start (via `mlflow.set_experiment(...)`
before `start_run()`), rather than accumulating everything
undifferentiated under `Default` the way this lab explicitly says to
avoid disturbing.

### 2.2 Experiment-level metadata and run-level metadata are scoped to different things — same "parameter vs. metric vs. artifact" discipline, one level up

```text
Experiment tag:  team=ml-platform    → describes the PROJECT/GROUPING as a whole
Run tag:          (not used here)     → would describe ONE SPECIFIC EXECUTION
```

Setting `team=analytics` on a *run* inside `churn-prediction` would
satisfy neither this lab's literal requirement nor its actual intent
— the requirement is about who owns the *project category*, not about
annotating any single training execution. This is structurally the
same distinction Day 21 drew between parameters, metrics, and
artifacts (each scoped to a run) — tags simply add a *fourth* scoped
category, and this lab's twist is that the scope moves up from "run"
to "experiment."

### 2.3 Inspect existing resources before creating new ones — same discipline as every prior "don't disturb what already exists" lab

```text
Default         ← seeded, pre-existing — DO NOT MODIFY
legacy-models   ← seeded, pre-existing — DO NOT MODIFY
fraud-detection, churn-prediction  ← the only two resources this lab should touch
```

Opening the Experiments page and inspecting what already exists
*before* creating anything new establishes a baseline to compare
against at the end — the same principle as Day 18's "verify
restoration against the exact baseline hash" and Day 19's "don't
touch the staged file a checker still expects" (both MLOps), just
applied here to not accidentally editing a seeded experiment's
metadata instead of a freshly-created one.

### 2.4 A description is attached to the experiment resource itself, not to any run inside it

```text
fraud-detection's description: "Machine learning project for detecting fraudulent transactions."
```

This text exists at the experiment level and is visible regardless of
how many runs exist inside it (including zero) — it's documentation
*about the project*, independent of any training activity. This is
why the lab can require a description on `fraud-detection` while
explicitly not requiring one on `churn-prediction`: each experiment's
metadata is set and verified completely independently of the other.

### 2.5 Verifying the correct experiment name before editing its metadata prevents a specific, easy-to-make mistake

```text
Intend to tag:  churn-prediction
Actually open:   Default            ← WRONG experiment, tag lands on the wrong resource
```

Because both experiments are created via the same UI flow and tagged
through the same kind of metadata control, it's easy to lose track of
*which* experiment is currently open when editing tags back-to-back —
re-confirming the experiment name immediately before saving a tag is
the specific guard against silently tagging the wrong resource, the
same exact-target discipline as every `--filters`/exact-path check
throughout the Cloud-AWS and DevOps sides of this series, here applied
by reading the UI carefully rather than passing a CLI flag.

### 2.6 Verification means checking the tag against the *specific* experiment, not just that *some* experiment somewhere has it

```text
Experiments list:
  fraud-detection    team: ml-platform   ← must be attached to THIS row
  churn-prediction   team: analytics     ← must be attached to THIS row
```

Confirming `team: ml-platform` appears *somewhere* on the Experiments
page isn't sufficient — it has to appear specifically next to
`fraud-detection`, and `team: analytics` specifically next to
`churn-prediction`. This is the same "verify the decisive, specific
field" pattern as every `--query`/JMESPath projection in the
Cloud-AWS CLI labs, just read off a UI table instead of extracted
programmatically.

### 2.7 The compressed reasoning chain

```text
Requirement: two new tagged experiments, Default/legacy-models untouched
   → open MLflow UI, inspect EXISTING experiments first — establish the baseline (§2.3)
   → create fraud-detection, with its required non-empty description (§2.4)
   → open fraud-detection SPECIFICALLY, add tag team=ml-platform at the EXPERIMENT level,
     not on any run (§2.2, §2.5)
   → create churn-prediction (description optional)
   → open churn-prediction SPECIFICALLY (re-confirm the name before editing!), add
     tag team=analytics (§2.5)
   → verify EACH tag against its SPECIFIC experiment row on the Experiments list (§2.6)
   → re-check Default and legacy-models are unchanged from the baseline (§2.3)
```

---

## 3. Concepts (reference)

### 3.1 MLflow experiment
A logical grouping for related runs — training jobs, their
parameters, metrics, and artifacts. Experiments let an organization
separate tracking by project rather than accumulating everything
under one undifferentiated bucket.

### 3.2 Experiment-level metadata: description and tags
Description is free-text documentation attached to the experiment as
a whole. Tags are key-value pairs used to classify or identify
ownership of the experiment itself — both exist independently of any
run inside it.

### 3.3 Experiment tag vs. run tag
An experiment tag classifies the project/grouping as a whole; a run
tag describes one specific execution. The two are stored and queried
independently — setting one does not set or imply the other.

### 3.4 Seeded/reference resources
Pre-existing experiments (`Default`, `legacy-models`) that a task may
require left unmodified — establishing their current state before
starting work is how you later confirm they weren't accidentally
touched.

### 3.5 Exact-resource verification
Confirming a piece of metadata exists is insufficient without
confirming it's attached to the *specific* resource the requirement
names — the same discipline as exact-path/exact-filter verification
throughout this series, applied here to UI-based experiment tagging.

---

## 4. Runbook

### 4.1 Open the MLflow UI and inspect existing experiments
```text
Open the MLflow UI (port 5000, from Day 20)
  → Experiments page
  → inspect Default and legacy-models — record their current state as the baseline
```

### 4.2 Create the `fraud-detection` experiment
```text
Experiments page → Create
  → Name: fraud-detection
  → Description: Machine learning project for detecting fraudulent transactions.
  → Create
```

### 4.3 Tag `fraud-detection` at the experiment level
```text
Open fraud-detection → experiment metadata controls
  → Key: team
  → Value: ml-platform
  → Save
```
Verify: Experiments list shows `team: ml-platform` next to
`fraud-detection` specifically.

### 4.4 Create the `churn-prediction` experiment
```text
Experiments page → Create
  → Name: churn-prediction
  → (description left blank — optional)
  → Create
```

### 4.5 Tag `churn-prediction` at the experiment level
```text
Confirm you are viewing churn-prediction (not Default or another experiment)
  → experiment metadata controls
  → Key: team
  → Value: analytics
  → Save
```

### 4.6 Final verification
```text
Experiments list → confirm:
  - fraud-detection:  non-empty description, team: ml-platform
  - churn-prediction: team: analytics
  - Default and legacy-models: unchanged from the §4.1 baseline
```

---

## 5. Final state

| Requirement | Result |
|---|---|
| `fraud-detection` experiment created | ✅ |
| `fraud-detection` description set (non-empty) | ✅ |
| `fraud-detection` tagged `team=ml-platform` | ✅ |
| `churn-prediction` experiment created | ✅ |
| `churn-prediction` tagged `team=analytics` | ✅ |
| `Default` unchanged | ✅ |
| `legacy-models` unchanged | ✅ |

```text
MLflow Tracking Server (localhost:5000)
 ├── Default            (unchanged)
 ├── legacy-models       (unchanged)
 ├── fraud-detection      — "Machine learning project for detecting fraudulent transactions."
 │                           tag: team=ml-platform
 └── churn-prediction     — tag: team=analytics
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Description shows `-` / empty | Saved against the wrong experiment, or left blank by mistake | Open the correct experiment, re-save a non-empty description |
| Tag appears on the wrong experiment | Lost track of which experiment was open while editing | Re-check the experiment name before every tag save (§2.5) |
| Tag exists on a run but not the experiment | Added via run-level metadata controls instead of experiment-level | Add it through the experiment's own metadata controls (§2.2) |
| New experiment missing from the list | Creation didn't complete, or exact name differs from required | Re-check the Experiments list for the exact name |
| `Default` or `legacy-models` accidentally modified | Edited the wrong experiment's metadata by mistake | Compare against the pre-task baseline (§4.1); restore if needed |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.7) out loud
      from "inspect existing experiments" to "verify both new tags
      against their specific experiment rows."
- [ ] Explain, in one sentence, why setting `team=analytics` on a run
      inside `churn-prediction` would not satisfy this lab's
      requirement.
- [ ] Explain why inspecting `Default` and `legacy-models` *before*
      creating anything new matters for later verification.
- [ ] Explain the difference between an experiment description and an
      experiment tag — what is each actually for?
- [ ] Explain the specific mistake that re-confirming the experiment
      name before saving a tag is meant to prevent.
- [ ] From memory, create a third experiment, give it a description
      and a `team` tag, log one run into it using
      `mlflow.set_experiment(...)` before `start_run()`, and confirm
      the run lands in the correct experiment — without copying any
      step from this file.

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
