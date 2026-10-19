# Day 25 — Register, Version, and Manage Model Lifecycle

A KodeKloud "100 MLOps" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each action exists and *how you'd derive it yourself*.

Day 22 established that experiment-level metadata (description, tags)
and run-level metadata are two different scopes, easy to confuse. This
lab hits the *exact same* scoping mistake one level up the MLflow
object hierarchy: a registered model's description and an individual
version's description are separately stored, and editing one has zero
effect on the other — which is precisely what failed the first
submission attempt here.

---

## 1. Scenario

Register two already-trained model artifacts under one registered
model `fraud-detector`, with a parent-level description, and assign
role-identifying aliases to each version.

| Requirement | Value |
|---|---|
| Registered model | `fraud-detector` |
| Registered-model description | contains "fraud" |
| Version 1 | source run `baseline`, alias `challenger` |
| Version 2 | source run `improved`, alias `champion` |

```text
Experiment: fraud-detection
 ├── run: baseline   (f1=0.80, n_estimators=100, max_depth=5)
 │        └── logged model ──register──▶ fraud-detector  Version 1 ── alias: challenger
 └── run: improved   (f1=0.89, n_estimators=200, max_depth=10)
          └── logged model ──register──▶ fraud-detector  Version 2 ── alias: champion

fraud-detector (registered model)
   description: "Fraud detection model for xFusionCorp transactions."
   ← set on the PARENT page, NOT on either version page
```

---

## 2. Reasoning model — how to *derive* the steps, not memorize them

### 2.1 Four distinct objects, one level deeper than Day 21/22's run/experiment pair

```text
Run (baseline, improved)        ← Day 21: where params/metrics/artifacts live
Logged model                     ← the artifact itself, tied to ONE run
Registered model (fraud-detector) ← a NEW, separate registry entry
Model version (1, 2)              ← a numbered snapshot WITHIN the registered model
```

Registering a logged model doesn't mutate the run or the logged
model — it creates a *new*, separate registry object (or, for a
second artifact, a new *version* under an existing one) that
references back to its source run. This is the same "creating a
reference vs. the thing it references" pattern as Day 18's Git tags —
the registered model version doesn't *contain* the training run, it
*points at* it.

### 2.2 Registration order determines version numbering — sequential, not based on any metric

```text
Register baseline FIRST  → becomes Version 1
Register improved SECOND → becomes Version 2
```

Version numbers are assigned strictly in the order artifacts are
registered under a given name — nothing about F1 score, training
date, or any other property influences this. Registering `improved`
first would make it Version 1 despite the task requiring it as
Version 2, purely because of ordering — the same "a derived value is
a consequence of the exact steps taken, not of some independent
correctness judgment" lesson as Day 19 MLOps's pipeline-contract
mismatch (the script writes what it writes; the declaration must
match it, not the other way around).

### 2.3 "Create New Model" vs. "register to an existing model" — same action, different target, easy to get backwards

```text
Register baseline  → "Create New Model" → fraud-detector now EXISTS (Version 1)
Register improved   → select EXISTING fraud-detector → becomes Version 2 of the SAME model
```

Picking "Create New Model" a second time would produce a *second*,
separate registered model (perhaps also implicitly named
`fraud-detector` or rejected as a duplicate, depending on the UI) —
not a second version of the one the task requires. The two
registration actions are superficially identical clicks but target
fundamentally different outcomes depending on which option is
selected at that one step.

### 2.4 Registered-model description and version description are two completely separate metadata fields — the exact Day 22 mistake, one level up

```text
fraud-detector                      (PARENT — the registered model itself)
  description: ???                    ← THIS is what the evaluator checked
  │
  ├── Version 1  description: ???    ← a SEPARATE field, visible on ITS OWN page
  └── Version 2  description: ???    ← ALSO separate
```

Setting a description on Version 1's page only ever affects Version
1's own metadata — it has no mechanism to propagate up to the parent
`fraud-detector` registered-model page, which maintains its own
independent description field. This is structurally identical to Day
22's experiment-tag-vs-run-tag confusion: a correctly-worded piece of
metadata, saved to the *wrong scope*, satisfies nothing about the
scope the requirement actually names.

### 2.5 Reading the evaluator's exact wording tells you exactly which scope failed

```text
"The 'fraud-detector' registered model has no description."
```

The error specifically names the *registered model*, not "a version"
or "Version 1" — this phrasing is the direct signal that the problem
is scope, not content (the description text itself was fine on the
version pages). This is the same "read the actual error before
guessing a fix" discipline as Day 24 MLOps's `TypeError` on
`search_logged_models` — the fix was findable directly from what the
message said, not from re-trying unrelated changes.

### 2.6 An alias is a mutable pointer to a version — not a deployment action

```text
Version 1 ← alias "challenger"    (just a LABEL pointing at this version)
Version 2 ← alias "champion"       (just a LABEL pointing at this version)
```

Assigning an alias changes nothing about whether a model is deployed,
serving traffic, or being used anywhere — it's purely a
human/tool-friendly name that *could* later be used by downstream
code to reference "whichever version currently holds `champion`"
instead of hardcoding a version number. This is the same "identity
vs. authorization/action" split recurring throughout this series (Day
18's Git tag, Day 20 AWS's `create-role`) — the alias exists, but
nothing about creating it causes anything else to happen.

### 2.7 Verifying each version's *source run* is a separate check from verifying its alias

```text
Version 1: source run = baseline?   ← one fact
Version 1: alias = challenger?       ← a DIFFERENT fact
```

Confirming a version has the right alias doesn't confirm it was
registered from the correct run, and vice versa — both need checking
independently, the same two-independent-facts pattern as Day 22's
experiment-name-vs-tag-value verification and every exact-resource
check throughout this series.

### 2.8 The compressed reasoning chain

```text
Requirement: fraud-detector registered model, described, with 2 correctly-ordered,
             correctly-aliased versions
   → open fraud-detection experiment, inspect baseline AND improved runs first (metrics/params)
   → register BASELINE FIRST, "Create New Model" → becomes Version 1 (§2.2, §2.3)
   → register IMPROVED SECOND, select the EXISTING fraud-detector → becomes Version 2 (§2.2, §2.3)
   → open the PARENT fraud-detector page (NOT a version page) → edit ITS description (§2.4)
   → open Version 1 → confirm source run = baseline → add alias "challenger" (§2.7)
   → open Version 2 → confirm source run = improved → add alias "champion" (§2.7)
   → if an evaluator complains "registered model has no description" — re-check
     SCOPE first, not content (§2.5)
```

---

## 3. Concepts (reference)

### 3.1 Logged model vs. registered model vs. model version
A logged model is an artifact tied to one specific run. A registered
model is a separate, named registry entry. A model version is a
numbered snapshot within a registered model, referencing back to the
logged model (and its source run) it was created from.

### 3.2 Version numbering
Assigned strictly by registration order under a given registered-model
name — independent of any metric, date, or other property of the
underlying runs.

### 3.3 "Create New Model" vs. registering to an existing model
The same UI action (registering a logged model) branches into two
fundamentally different outcomes depending on whether a new
registered model is created or an existing one is targeted.

### 3.4 Registered-model description vs. version description
Two independently-stored metadata fields at different scopes in the
object hierarchy — editing one never updates the other, the same
scoping distinction as Day 22's experiment-tag vs. run-tag split.

### 3.5 Alias as a mutable pointer
A human-readable name attached to a specific model version,
re-pointable to a different version later, with no inherent
deployment or serving behavior of its own.

### 3.6 Reading evaluator/error messages for scope, not just content
An error naming a specific object (e.g. "the registered model") is
telling you exactly which scope to re-check — correct content saved
to the wrong scope still fails the check.

---

## 4. Runbook

### 4.1 Open the experiment and inspect both runs
```text
MLflow UI → fraud-detection experiment
  → baseline:  f1_score=0.80, n_estimators=100, max_depth=5
  → improved:  f1_score=0.89, n_estimators=200, max_depth=10
```

### 4.2 Register `baseline` as Version 1
```text
Open baseline run → Overview → Logged models → click the model name
  → Register model → Create New Model → name: fraud-detector → Register
```
Confirm it becomes **Version 1**, source run `baseline`.

### 4.3 Register `improved` as Version 2
```text
Open improved run → Overview → Logged models → click the model name
  → Register model → select EXISTING fraud-detector (do NOT create a new model)
  → Register
```
Confirm it becomes **Version 2**, source run `improved`.

### 4.4 Set the description on the PARENT registered model
```text
Registered Models → fraud-detector (the PARENT page, not a version page)
  → Description → Edit
  → "Fraud detection model for xFusionCorp transactions."
  → Save
```
URL pattern: `#/models/fraud-detector` (parent) vs.
`#/models/fraud-detector/versions/1` (version — do NOT edit here).

### 4.5 Add the `challenger` alias to Version 1
```text
fraud-detector → Version 1 → confirm source run = baseline
  → Aliases → Add → challenger → Save
```

### 4.6 Add the `champion` alias to Version 2
```text
fraud-detector → Version 2 → confirm source run = improved
  → Aliases → Add → champion → Save
```

### 4.7 Final verification
```text
Registered Models → fraud-detector
  → description visible on THIS page: "Fraud detection model for xFusionCorp transactions."
  → Version 1: source=baseline, alias=challenger
  → Version 2: source=improved, alias=champion
```

---

## 5. Final state

| Requirement | Result |
|---|---|
| Registered model `fraud-detector` created | ✅ |
| Description set on the PARENT page, contains "fraud" | ✅ |
| Version 1 ← `baseline` run | ✅ |
| Version 1 alias `challenger` | ✅ |
| Version 2 ← `improved` run | ✅ |
| Version 2 alias `champion` | ✅ |

```text
fraud-detector  (registered model)
   description: "Fraud detection model for xFusionCorp transactions."
   │
   ├── Version 1 ── source: baseline run (f1=0.80)  ── alias: challenger
   └── Version 2 ── source: improved run (f1=0.89)  ── alias: champion
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Version numbers reversed (improved = V1) | `improved` was registered before `baseline` | Restart the registration if the lab allows it, baseline first |
| A second registered model appears | "Create New Model" selected a second time instead of targeting the existing one | Register the improved artifact against the existing `fraud-detector` entry |
| Evaluator: "registered model has no description" | Description was only saved on a version page | Edit the PARENT model page's Description specifically |
| Alias missing | Not saved, or added to the wrong version | Open the intended version, check its Aliases field directly |
| Wrong source run shown on a version | Wrong logged artifact was registered | Re-check which run's Logged models entry was clicked before registering |
| Artifact logged but not appearing in the registry | Logging and registering are separate actions | Use "Register model" explicitly on the logged model artifact |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.8) out
      loud from "inspect both runs" to "final verification on the
      parent page."
- [ ] Explain, in one sentence, why registering `improved` before
      `baseline` would produce the wrong version numbering.
- [ ] Explain why a description saved on Version 1's page doesn't
      satisfy a check for the registered model's description.
- [ ] Explain the difference between selecting "Create New Model" and
      selecting an existing registered model during the Register
      model flow.
- [ ] Explain why assigning the `champion` alias to Version 2 doesn't
      by itself deploy or serve that model.
- [ ] Explain how the evaluator's exact error wording
      ("the 'fraud-detector' registered model has no description")
      pointed directly at the scope of the actual mistake.
- [ ] From memory, register a third run as Version 3 under
      `fraud-detector`, assign it a new alias (e.g. `shadow`), and
      verify its source run and alias independently — without
      copying any step from this file.

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
