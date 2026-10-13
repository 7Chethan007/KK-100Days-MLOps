# Day 21 — Log an ML Experiment to MLflow

A KodeKloud "100 MLOps" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive it yourself*.

Day 20 stood up the tracking server with no ML content in it — a
service waiting to be used. This lab is that exact "future training
job" referenced in Day 20 §2.9: a Python script that actually calls
`set_tracking_uri`, `log_params`, `log_metric`, and
`mlflow.sklearn.log_model` against the server already running on
`localhost:5000`.

---

## 1. Scenario

Complete three TODO blocks in `/root/code/log_experiment.py`, run it,
and verify the resulting run is persisted in the `Default` experiment
on the MLflow server started in Day 20.

Recorded in the run:

| Category | Values |
|---|---|
| Parameters | `n_estimators=100`, `max_depth=5`, `random_state=42` |
| Metrics | `accuracy=0.75`, `f1_score≈0.8571` |
| Artifact | sklearn model, artifact path `model` |
| Run | `serious-goose-144`, status `Finished` |

```text
log_experiment.py
      │ set_tracking_uri("http://localhost:5000")
      ▼
MLflow Tracking API (Day 20's server)
      │
      ├──▶ backend store: params, metrics, run status
      └──▶ artifact store: serialized sklearn model
      ▼
MLflow UI → run serious-goose-144, Finished, model Ready
```

---

## 2. Reasoning model — how to *derive* the code, not memorize it

### 2.1 This script is the client side of Day 20's server/client split

```text
Day 20: deployed a SERVICE — nothing reports into it yet
Day 21: writes a CLIENT — the first thing that actually does
```

Day 20 §2.9 described, as a forward reference, exactly what a training
script needs to do to report into the tracking server:
`set_tracking_uri` + `log_*` calls inside `start_run()`. This lab is
that reference made concrete — nothing about the server changes; only
a client now exists that talks to it.

### 2.2 `localhost:5000` is correct here specifically because client and server are co-located

```text
Day 20: --host 0.0.0.0  → what the SERVER binds to, for the PROXY's benefit
Day 21: localhost:5000  → what the CLIENT connects to, since it runs on the SAME machine
```

These are two different addressing decisions solving two different
problems, even though they reference the same server. `localhost`
is correct *only* because the script and the server happen to run on
the same workstation — if training ran in a separate container or
different machine, `localhost` would resolve to the wrong host
entirely, and the actual reachable address (or service DNS name)
would be required instead.

### 2.3 Parameters recorded are not the same as parameters *used*

```text
params = {"n_estimators": 100, "max_depth": 5, "random_state": 42}
model = DummyClassifier(strategy="most_frequent")   ← none of params actually passed in
```

`mlflow.log_params(params)` only sends a dictionary of key-value pairs
to the tracking server — it has no mechanism to inspect or enforce
that the trained estimator actually used those values. Here the
`DummyClassifier` never receives `n_estimators`/`max_depth`/
`random_state` at all; they're tracking metadata the lab requires
recorded, not configuration that shaped this particular model. In a
real workflow, failing to actually pass logged parameters into the
estimator would silently produce a *misleading* record — the metadata
would claim a configuration the model never used.

### 2.4 `DummyClassifier(strategy="most_frequent")` makes the output fully derivable, not mysterious

```text
y_fit = [1, 1, 1, 0]   → most frequent label is 1
predict(X_fit)          → [1, 1, 1, 1]  (always predicts the majority class)
```

Comparing against actual labels `[1,1,1,0]`: 3 of 4 correct →
`accuracy = 0.75`. For F1: predicted-positive = 4, true positives = 3,
so precision = 3/4; actual positives = 3, all predicted positive, so
recall = 1; F1 = 2·(precision·recall)/(precision+recall) =
2·(0.75·1)/(0.75+1) ≈ 0.8571. Every number in this lab is a
deterministic consequence of a fixed baseline fixture, not a
meaningful signal about model quality — the point is to exercise the
*logging* mechanics, not to evaluate anything.

### 2.5 `log_params(dict)` vs. multiple `log_param(key, value)` calls — same destination, different ergonomics

```text
mlflow.log_params({"a": 1, "b": 2})
# equivalent to:
mlflow.log_param("a", 1)
mlflow.log_param("b", 2)
```

Both land identical data in the backend store; `log_params` is purely
a convenience for when the values are already collected in a
dictionary, exactly the same "batch vs. individual calls, same
underlying effect" distinction as `dvc.yaml`'s declared `params:` list
vs. individually reading each value in Day 19/20's MLOps labs.

### 2.6 Metric *names* are the actual interface — a local variable name is irrelevant

```python
f1 = f1_score(y_fit, preds)
mlflow.log_metric("f1_score", f1)   # the STRING "f1_score" is what the UI/API key on
```

The Python variable `f1` is just local scratch space; what the lab
requirement and the MLflow UI actually care about is the string key
passed to `log_metric`. Naming the variable differently from the
logged key is completely fine — what would break the requirement is
logging under the wrong *string*, since that string is the only thing
MLflow persists and later queries against.

### 2.7 `with mlflow.start_run():` scopes which calls belong to which run

```text
with mlflow.start_run():
    mlflow.log_params(...)      ← associated with THIS run
    mlflow.log_metric(...)       ← associated with THIS run
    mlflow.sklearn.log_model(...) ← associated with THIS run
# exiting the block cleanly → run finalized, status Finished
```

Every logging call made while inside the context attaches to the one
active run; nothing needs to be told *which* run to log against
explicitly. This is why run name (`serious-goose-144`, auto-generated)
and Run ID (`d7bcfe0371c14158abf019f0ce745348`, the actual unique
identifier) are separate concepts — the name is a human-friendly label
assigned automatically, while the ID is what uniquely and permanently
identifies this specific run's record.

### 2.8 A deprecation warning is not a lab failure

```text
artifact_path is deprecated in favor of name
```

MLflow's own API evolving to prefer a different keyword
(`mlflow.sklearn.log_model(model, name="model")` in newer versions)
doesn't retroactively break the currently-installed version's
behavior — the warning is informational, and the lab's required
result (a logged model with status `Ready`) was still achieved using
`artifact_path`. The lesson generalizes: match the syntax to the
*installed* version actually in use, not to whatever the latest docs
recommend, and don't treat a warning as a failed check without
confirming it actually blocked anything.

### 2.9 End-to-end verification: script output, the returned link, and the UI are three independent confirmations

```text
stdout: "accuracy=0.75, f1_score=0.8571..."       → the SCRIPT computed and printed correctly
printed URL with the run ID                        → MLflow ACCEPTED and recorded the run
UI shows params/metrics/model, status Finished     → the FULL round trip, visually confirmed
```

Each of these checks a different point in the chain — a script
printing correct numbers doesn't by itself prove MLflow received and
stored them; MLflow accepting the run doesn't by itself prove the UI
renders it correctly. This is the same layered-verification discipline
as Day 20 §2.8 (pgrep/log/curl/browser), just applied to a logging
client instead of the server itself.

### 2.10 A logged model artifact is not a registered model

```text
Logged model (this lab):  a file recorded against ONE run — "Ready" means recorded correctly
Registered model (NOT this lab): a separately-managed entity in the Model Registry,
                                   tracking versions/lifecycle stages across MANY runs
```

"Ready" in the run's Artifacts/Logged models view confirms the model
object was successfully serialized and stored — it says nothing about
deployment, inference readiness, or registry lifecycle management,
which are deliberately separate MLflow concepts this lab doesn't touch.

### 2.11 The compressed reasoning chain

```text
Requirement: log one training run into Day 20's tracking server
   → mlflow.set_tracking_uri("http://localhost:5000")  — correct because co-located (§2.2)
   → build params dict — metadata to RECORD, not necessarily values the model actually used (§2.3)
   → fit a deterministic baseline (DummyClassifier) — output fully derivable by hand (§2.4)
   → compute accuracy/f1 from actual predictions, not hardcoded values
   → with mlflow.start_run():  — scopes every call below to ONE run (§2.7)
   →   log_params(params)  /  log_metric(exact_string_key, value)  (§2.5, §2.6)
   →   mlflow.sklearn.log_model(model, artifact_path="model")  — version-appropriate kwarg (§2.8)
   → verify via THREE independent signals: stdout, returned run URL, UI (§2.9)
   → remember: "Ready" means logged correctly, NOT registered/deployed (§2.10)
```

---

## 3. Concepts (reference)

### 3.1 Parameter vs. metric vs. artifact
A parameter is a configuration value chosen for a run; a metric is a
numerical result measured from that run; an artifact is a file or
structured output the run produced. A single MLflow run ties all
three categories together as one record.

### 3.2 Run vs. experiment
A run is one execution; an experiment groups related runs for
comparison. This lab's run belongs to the `Default` experiment
(experiment ID `0`), since no custom experiment was created.

### 3.3 Run name vs. Run ID
The run name (`serious-goose-144`) is an auto-generated, human-
readable label — convenient but not guaranteed unique in the way an
identifier needs to be. The Run ID is the actual unique identifier
MLflow uses internally and in URLs.

### 3.4 `mlflow.start_run()` context scoping
Logging calls made inside the `with mlflow.start_run():` block attach
automatically to that run; no explicit run reference needs to be
passed to each `log_*` call.

### 3.5 Logged model vs. registered model
A logged model is an artifact recorded against a specific run. A
registered model is a separate Model Registry entity tracking
versions and lifecycle stages across potentially many runs. Logging
doesn't imply or trigger registration.

### 3.6 Recorded parameters vs. parameters actually applied to the estimator
Logging a parameter is purely a bookkeeping action — it has no
mechanism to verify, or enforce, that the trained model's own
configuration matches what was logged. These can diverge, as they
deliberately do in this lab's fixture.

---

## 4. Runbook

### 4.1 The completed script
`/root/code/log_experiment.py`:
```python
"""MLflow experiment logging — record a training run."""
import numpy as np
import mlflow
import mlflow.sklearn
from sklearn.dummy import DummyClassifier
from sklearn.metrics import accuracy_score, f1_score

mlflow.set_tracking_uri("http://localhost:5000")

params = {
    "n_estimators": 100,
    "max_depth": 5,
    "random_state": 42,
}

X_fit = np.array([[0.0], [1.0], [2.0], [3.0]])
y_fit = np.array([1, 1, 1, 0])

model = DummyClassifier(strategy="most_frequent").fit(X_fit, y_fit)

preds = model.predict(X_fit)
accuracy = accuracy_score(y_fit, preds)
f1 = f1_score(y_fit, preds)

with mlflow.start_run():
    mlflow.log_params(params)
    mlflow.log_metric("accuracy", accuracy)
    mlflow.log_metric("f1_score", f1)
    mlflow.sklearn.log_model(model, artifact_path="model")
    print(f"accuracy={accuracy}, f1_score={f1}")
```

### 4.2 Run the script
```bash
python /root/code/log_experiment.py
```
```text
accuracy=0.75, f1_score=0.8571428571428571
http://localhost:5000/#/experiments/0/runs/d7bcfe0371c14158abf019f0ce745348
```
Run ID: `d7bcfe0371c14158abf019f0ce745348`.

### 4.3 Verify in the MLflow UI
Open the printed URL (or navigate: Experiments → Default → the run).
Confirmed:
- Run name: `serious-goose-144`
- Experiment ID: `0`
- Status: `Finished`
- 3 parameters, 2 metrics
- 1 logged model (`model`), status `Ready`

---

## 5. Final state

| Requirement | Result |
|---|---|
| `set_tracking_uri` points at Day 20's running server | ✅ |
| 3 parameters logged (`n_estimators`, `max_depth`, `random_state`) | ✅ |
| 2 metrics logged (`accuracy`, `f1_score`) with exact required names | ✅ |
| Model artifact logged, status `Ready` | ✅ |
| Run finalized with status `Finished` | ✅ |
| Verified via script output, returned run URL, and UI | ✅ |

```text
localhost:5000 (Day 20's MLflow server)
 └── Default experiment (id: 0)
       └── run: serious-goose-144  (d7bcfe03...)
              ├── params: n_estimators=100, max_depth=5, random_state=42
              ├── metrics: accuracy=0.75, f1_score=0.8571...
              └── model artifact: "model" — Ready
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Script can't connect to the tracking server | Wrong `tracking_uri`, or Day 20's server isn't running | Confirm `pgrep -af 'mlflow server'`; re-check the URI matches the actual bind address/port |
| Metrics show up under an unexpected name in the UI | Logged under a different string key than required | Check the first argument to `log_metric` exactly, not just the local variable name (§2.6) |
| `log_model` emits a deprecation warning | Installed MLflow version prefers `name=` over `artifact_path=` | Non-fatal; use the syntax your installed version documents (§2.8) |
| Run never appears as `Finished` | Exception raised inside the `with mlflow.start_run():` block before it exited cleanly | Check the script's traceback; fix the error, rerun |
| Parameters logged don't match what the model actually used | Expected in this lab's fixture (§2.3) — but a real bug in production code | Pass the same values into both the estimator and `log_params`, don't let them drift independently |
| Can't find the run afterward | Looked in the wrong experiment, or lost the Run ID | Default experiment (id `0`) unless explicitly set otherwise; capture the returned run URL at logging time |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.11) out loud
      from "`set_tracking_uri`" to "verified via script output, run
      URL, and UI."
- [ ] Explain, in one sentence, why `localhost:5000` is correct here
      but would be wrong if the training script ran in a separate
      container from the tracking server.
- [ ] By hand, recompute `accuracy` and `f1_score` for
      `y_fit=[1,1,1,0]` against a classifier that always predicts `1`
      — confirm it matches `0.75`/`≈0.8571`.
- [ ] Explain why the `params` dictionary values aren't actually
      passed into `DummyClassifier(...)`, and why that's acceptable
      for this lab's purpose but would be a real bug elsewhere.
- [ ] Explain the difference between a run's auto-generated name and
      its Run ID — which one would you use to reliably look the run
      up later?
- [ ] Explain why a "Ready" logged model is not the same as a
      registered model.
- [ ] From memory, modify the script to log a second run with
      different `params` values (actually passed into the model this
      time), then compare both runs' metrics in the UI — without
      copying any command from this file.

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
