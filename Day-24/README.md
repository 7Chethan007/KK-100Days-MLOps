# Day 24 — Enable MLflow Autologging

A KodeKloud "100 MLOps" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive it yourself*.

Day 21 logged a run by hand: explicit `log_params`, `log_metric`,
`log_model` calls, one per fact the lab wanted recorded. This lab
removes that manual work entirely for a supported framework
(scikit-learn) — `mlflow.sklearn.autolog()` observes the training call
itself and captures parameters, metrics, and the model without a
single explicit `log_*` call. Day 22's experiment-creation mechanics
(`set_experiment`) are reused here exactly as before, just now paired
with autologging instead of manual logging.

---

## 1. Scenario

Complete two TODOs in `/root/code/autolog_experiment.py` so that
fitting a `LogisticRegression` estimator automatically produces a
fully-tracked MLflow run — no manual `log_param`/`log_metric`/
`log_model` calls — inside a new experiment named `autolog-demo`.

| Requirement | Expected state |
|---|---|
| Tracking server | `http://localhost:5000` (from Day 20) |
| Experiment | `autolog-demo` |
| Framework / estimator | scikit-learn `LogisticRegression` |
| Run | at least one completed run |
| Parameters | constructor parameters, including defaults |
| Metrics | automatically logged training metrics |
| Model artifacts | `MLmodel` + serialized estimator |
| Code changes to the model | not required |

```text
set_tracking_uri("http://localhost:5000")
        │
        ▼
mlflow.sklearn.autolog()          ← must come BEFORE .fit()
        │
        ▼
set_experiment("autolog-demo")
        │
        ▼
model.fit(X, y)   ← the event autolog is watching for
        │
        ▼
MLflow auto-creates a run, captures:
   params (14) · metrics (7) · logged model (MLmodel + model.pkl)
```

---

## 2. Reasoning model — how to *derive* the code, not memorize it

### 2.1 Four separate responsibilities, each answering a different question

```text
set_tracking_uri()  → WHERE should tracking data go?
set_experiment()     → WHICH experiment should receive the run?
autolog()            → WHAT should MLflow capture automatically?
model.fit()          → the actual training operation being observed
```

These are four independent configuration/execution steps, not one
combined "turn on tracking" switch. Keeping them separate in your head
is what makes debugging tractable later: if a run lands in the wrong
experiment, that's a `set_experiment` ordering problem; if parameters
are missing, that's an `autolog()` ordering problem — they're never
the same root cause.

### 2.2 `autolog()` works by *instrumenting* `.fit()` — it has to be armed before that call happens

```text
1. mlflow.sklearn.autolog()   ← arms the instrumentation
2. model = LogisticRegression(...)
3. model.fit(X, y)             ← the INSTRUMENTED call — THIS is what autolog observes
```

Autologging doesn't retroactively scan a model that's already been
trained — it wraps/instruments the scikit-learn training API itself,
so it can only capture information from a `.fit()` call that happens
*after* instrumentation is armed. Calling `autolog()` after `.fit()`
has already run is exactly like installing a security camera after
the event you wanted it to record — nothing to capture anymore. This
is the single most important ordering rule in the entire lab.

### 2.3 `set_experiment()` only decides *where* the next run goes — same separation as Day 22's experiment vs. run distinction

```text
set_experiment("autolog-demo")   → creates it if missing, selects it as the TARGET
                                     for whatever run gets created next
```

This call trains nothing and logs nothing by itself — exactly the
same "identity vs. content" split Day 22 drew between creating an
experiment and logging a run into it. Calling it *after* training has
already started would be too late for the same reason as §2.2: the
run would already be associated with whatever experiment was active
(likely `Default`) before `set_experiment` ran.

### 2.4 Explicitly-supplied constructor arguments are a small subset of the estimator's real configuration

```python
model = LogisticRegression(C=1.0, max_iter=100, random_state=42)
```
```text
Only 3 keyword arguments supplied...
...but the estimator ALSO has: solver, tol, fit_intercept, class_weight,
   dual, l1_ratio, n_jobs, penalty, verbose, warm_start, intercept_scaling
```

Every constructor argument not explicitly passed still has a value —
it comes from scikit-learn's own default. Two training runs that
supply identical explicit arguments could still behave differently if
run against different library versions with different defaults for
the *unsupplied* ones. Autologging captures the estimator's *full*
effective configuration (all 14 parameters observed in this lab), not
just the 3 the script happened to type out — which is exactly the gap
manual logging would leave unless someone remembered to log every
single parameter by hand.

### 2.5 A toy, deterministic dataset isolates "does the logging pipeline work" from "is this a good model"

```python
X = [[0,0],[0,1],[1,0],[1,1]]
y = [0,0,1,1]
```

This 4-row dataset has no relationship to a real fraud/churn/etc.
dataset — its only job is to be valid input to `.fit()` so the
*tracking mechanics* can be exercised and verified. The resulting
"perfect" metrics (`training_accuracy_score = 1`, etc.) are an
artifact of fitting a model to 4 trivially-separable points, not
evidence about real-world model quality — the same caution Day 21
raised about its own `DummyClassifier` fixture, just now with a real
(if tiny) `LogisticRegression`.

### 2.6 Autologging can create a run for you — no `with mlflow.start_run():` block required

```text
Day 21: explicit  with mlflow.start_run():  ...  (manual scope)
Day 24: mlflow.sklearn.autolog() + model.fit(...)  → MLflow opens/closes the run itself
```

Because the scikit-learn integration observes `.fit()` directly, it
can start a run automatically if none is already active, log
everything it captures into that run, and finish it when training
completes — eliminating the explicit run-management block entirely.
This is the concrete payoff of autologging: not just skipping
individual `log_*` calls, but skipping the run lifecycle management
too.

### 2.7 A toolchain's own API can legitimately require an argument the docs/habit didn't expect — verify by reading the actual error

```text
client.search_logged_models(experiment_ids=["1"])
                              ^^^^^^^^^^^^^^^^^^^^
TypeError: missing 1 required positional argument: 'experiment_ids'
```

The first attempt to call `search_logged_models()` without an
argument failed with a specific, readable error naming exactly what
was missing. The fix wasn't guesswork — it was reading the error,
supplying the named required argument (the experiment ID, as a list),
and retrying. This is the same "read the actual error/output before
changing anything" discipline running through every lab in this
series, here applied to a Python API instead of a CLI tool or
service.

### 2.8 In MLflow 3, a logged model can live *separately* from the run's ordinary artifacts — checking one doesn't rule out the other

```text
client.list_artifacts(run_id)         → estimator.html, 3 PNG plots — NO MLmodel, NO model.pkl
client.search_logged_models(...)      → finds the model as its OWN tracked entity
  → model.artifact_location           → THIS directory actually contains MLmodel + model.pkl
```

An ordinary run-artifact listing coming back without `MLmodel`/
`model.pkl` is not evidence that model logging failed — in MLflow 3,
a model can be tracked as a first-class "logged model" entity
associated with, but stored separately from, the run's regular
artifact folder. The decisive check is `search_logged_models`
followed by listing *that* model's own artifact location — the same
"check the decisive, specific location, not just the first place you
looked" pattern as every exact-verification lesson in this series
(e.g. Day 20 AWS's `list-attached-role-policies` being decisive over
`get-role` alone).

### 2.9 HTTP reachability, experiment existence, run creation, parameter/metric capture, and model persistence are five separate claims

```text
curl -I http://localhost:5000        → is the endpoint reachable at all?
MLflow UI → autolog-demo exists?      → did set_experiment actually create/select it?
run visible under that experiment?    → was a run actually created and associated correctly?
parameters/metrics panel populated?   → did autolog actually CAPTURE training info?
search_logged_models + artifact list? → was the MODEL specifically persisted?
```

None of these checks implies any of the others — a `200 OK` from
`curl` says nothing about whether autologging captured anything; a
visible run says nothing about whether its parameters panel is
populated; a populated parameters panel says nothing about whether
the model itself was persisted. This is the same layered,
independent-verification discipline as Day 20's pgrep/log/curl/browser
chain, now extended to five distinct checkpoints because this lab has
one more moving part (the automatic instrumentation layer) than a
plain server-start.

### 2.10 A warning is not automatically a failure — read what it's actually warning about

```text
Pickle security warning   → about DESERIALIZING untrusted pickle files later, not about THIS run failing
FutureWarning (sklearn)    → about a future API change in a LATER version, not about THIS run failing
```

Both warnings appeared and neither prevented the run from completing
successfully — the correct response was to understand what each
warning is actually flagging (a security consideration for consumers
of the pickle file; a forward-compatibility notice from a dependency)
rather than reflexively treating any non-empty stderr output as proof
something went wrong.

### 2.11 The compressed reasoning chain

```text
Requirement: fully automatic MLflow tracking for a LogisticRegression fit, in autolog-demo
   → set_tracking_uri("http://localhost:5000")   — WHERE (§2.1)
   → mlflow.sklearn.autolog()                     — WHAT to capture, MUST precede .fit() (§2.1, §2.2)
   → mlflow.set_experiment("autolog-demo")        — WHICH experiment, also before .fit() (§2.3)
   → construct LogisticRegression(C=1.0, max_iter=100, random_state=42)
   → model.fit(X, y)   — the instrumented event; autolog auto-creates the run (§2.6)
   → verify in 5 INDEPENDENT layers, not one:
       curl reachability → experiment exists → run exists → params/metrics populated →
       search_logged_models + its OWN artifact listing for MLmodel/model.pkl (§2.8, §2.9)
   → a TypeError naming a missing argument → fix by reading the error, not by guessing (§2.7)
   → warnings (pickle, FutureWarning) ≠ failure — read what each is actually about (§2.10)
```

---

## 3. Concepts (reference)

### 3.1 Autologging
A framework-specific MLflow integration that instruments a training
API (here, scikit-learn's `.fit()`) so that supported information —
parameters, metrics, the trained model — is captured automatically
without explicit `log_*` calls, provided the integration was enabled
*before* the instrumented call executed.

### 3.2 `set_tracking_uri` / `set_experiment` / `autolog` / `fit` — four distinct responsibilities
Where tracking data goes, which experiment a run belongs to, what
gets captured automatically, and the actual training event itself are
four separable concerns — mixing up their ordering is the most common
way autologging silently fails to capture what's expected.

### 3.3 Explicit arguments vs. effective configuration
An estimator's full configuration includes every constructor
parameter, whether explicitly supplied or left at its library default.
Autologging captures the full effective configuration, not just
whatever subset of arguments a script happened to type out.

### 3.4 Run-managed lifecycle under autologging
When a supported training call occurs with no explicitly active run,
the autologging integration can open and close a run automatically —
removing the need for an explicit `with mlflow.start_run():` block.

### 3.5 Logged model as a separate entity (MLflow 3)
A trained model can be tracked as its own entity (with its own ID and
artifact location) associated with a run, distinct from that run's
ordinary artifact folder. Checking only the run's regular artifacts
can miss the model entirely even when it was logged correctly.

### 3.6 Reading an error vs. guessing a fix
A specific, well-formed error (e.g. a Python `TypeError` naming a
missing argument) is direct, actionable information — the correct
response is to supply exactly what the error says is missing, not to
try unrelated changes.

### 3.7 Warnings vs. failures
A warning communicates a condition worth being aware of; it does not
by itself mean an operation failed. Distinguishing the two requires
reading what the specific warning is actually about, not just noting
that output appeared.

---

## 4. Runbook

### 4.1 Inspect the provided scaffold
```bash
cd /root/code
cat -n autolog_experiment.py
```
Confirms the imports, the pre-set tracking URI, and locates the two
TODO blocks.

### 4.2 Complete the two TODOs
```python
mlflow.set_tracking_uri("http://localhost:5000")

mlflow.sklearn.autolog()

mlflow.set_experiment("autolog-demo")

X = np.array([[0, 0], [0, 1], [1, 0], [1, 1]])
y = np.array([0, 0, 1, 1])

model = LogisticRegression(
    C=1.0,
    max_iter=100,
    random_state=42
)

model.fit(X, y)

print("Autolog run complete — check the MLflow UI")
```
Both `autolog()` and `set_experiment()` execute before `.fit()` —
the ordering rule from §2.2/§2.3.

### 4.3 Run the script
```bash
python /root/code/autolog_experiment.py
```
```text
Experiment with name 'autolog-demo' does not exist.
Creating a new experiment.

Created MLflow autologging run with ID
'b3143b8353214998a1c6f867a1c1285d'

🏃 View run resilient-vole-700 at:
http://localhost:5000/#/experiments/1/runs/b3143b8353214998a1c6f867a1c1285d

🧪 View experiment at:
http://localhost:5000/#/experiments/1

Autolog run complete — check the MLflow UI
```
A pickle security warning and a scikit-learn `FutureWarning` also
appeared — neither prevented completion (§2.10).

### 4.4 Verify the tracking server is reachable
```bash
curl -I http://localhost:5000
```
```text
HTTP/1.1 200 OK
server: uvicorn
content-type: text/html; charset=utf-8
```
Reachability only — not proof of successful tracking (§2.9).

### 4.5 Verify the experiment and run in the UI
```text
MLflow UI → autolog-demo (experiment ID: 1)
  → run: resilient-vole-700
      Run ID: b3143b8353214998a1c6f867a1c1285d
      Status: Finished
      Source: autolog_experiment.py
```

### 4.6 Verify the parameters
UI Parameters panel — 14 parameters, including:

| Parameter | Observed value |
|---|---|
| `C` | `1.0` |
| `class_weight` | `None` |
| `dual` | `False` |
| `fit_intercept` | `True` |
| `intercept_scaling` | `1` |
| `l1_ratio` | `0.0` |
| `max_iter` | `100` |
| `n_jobs` | `None` |
| `penalty` | `deprecated` |
| `random_state` | `42` |
| `solver` | `lbfgs` |
| `tol` | `0.0001` |
| `verbose` | `0` |
| `warm_start` | `False` |

11 of these 14 were never explicitly supplied by the script (§2.4).

### 4.7 Verify the metrics
UI Metrics panel — 7 metrics:

| Metric | Observed value |
|---|---:|
| `training_precision_score` | `1` |
| `training_recall_score` | `1` |
| `training_f1_score` | `1` |
| `training_accuracy_score` | `1` |
| `training_log_loss` | `0.5125920277987835` |
| `training_roc_auc` | `1` |
| `training_score` | `1` |

Perfect scores on a 4-row toy dataset — not evidence of real-world
model quality (§2.5).

### 4.8 Inspect ordinary run artifacts
```python
import mlflow

client = mlflow.tracking.MlflowClient(tracking_uri="http://localhost:5000")
run_id = "b3143b8353214998a1c6f867a1c1285d"

for artifact in client.list_artifacts(run_id):
    print(artifact.path, "directory" if artifact.is_dir else "file")
```
```text
estimator.html file
training_confusion_matrix.png file
training_precision_recall_curve.png file
training_roc_curve.png file
```
No `MLmodel`/`model.pkl` here — expected in MLflow 3 (§2.8), not a
failure signal.

### 4.9 Find the logged model (the decisive check)
First attempt, missing a required argument:
```python
models = client.search_logged_models()
```
```text
TypeError: MlflowClient.search_logged_models()
missing 1 required positional argument: 'experiment_ids'
```
Corrected call:
```python
models = client.search_logged_models(experiment_ids=["1"])

for model in models:
    if model.source_run_id == run_id:
        print("Model ID:", model.model_id)
        print("Artifact location:", model.artifact_location)

        artifacts = mlflow.artifacts.list_artifacts(
            artifact_uri=model.artifact_location,
            tracking_uri="http://localhost:5000",
        )
        for artifact in artifacts:
            print(artifact.path, "directory" if artifact.is_dir else "file")
```
```text
Model ID: m-af8690d081e2420f8fd63056a028eb51
Artifact location: /root/code/mlflow-artifacts/1/models/m-af8690d081e2420f8fd63056a028eb51/artifacts

artifacts/MLmodel file
artifacts/conda.yaml file
artifacts/model.pkl file
artifacts/python_env.yaml file
artifacts/requirements.txt file
```
Both required files (`MLmodel`, `model.pkl`) confirmed present — this
is the decisive model verification.

---

## 5. Final state

| Requirement | Evidence | Result |
|---|---|---|
| Tracking server reachable | HTTP `200 OK` | ✅ |
| Experiment `autolog-demo` exists | Experiment ID `1` | ✅ |
| Run created | Run ID `b3143b8353214998a1c6f867a1c1285d` | ✅ |
| Parameters auto-captured, including defaults | 14 parameters | ✅ |
| Metrics auto-captured | 7 metrics | ✅ |
| Logged model associated with the run | Model ID `m-af8690d081e2420f8fd63056a028eb51` | ✅ |
| `MLmodel` present | model artifact listing | ✅ |
| `model.pkl` present | model artifact listing | ✅ |
| No manual `log_*` calls needed | script contains none | ✅ |

```text
MLflow Tracking Server (localhost:5000)
 └── Experiment: autolog-demo (id: 1)
       └── Run: resilient-vole-144 (b3143b83...)
              ├── 14 parameters (3 explicit + 11 defaults)
              ├── 7 metrics (training_*)
              ├── run artifacts: estimator.html, 3 training plots
              └── Logged model: m-af8690d0...
                     └── artifacts/: MLmodel, model.pkl, conda.yaml,
                                      python_env.yaml, requirements.txt
```

---

## 6. Troubleshooting

| Symptom | Cause or interpretation | Resolution |
|---|---|---|
| Experiment doesn't exist | `set_experiment` never called | Call it before `.fit()` |
| Run appears under `Default` | `set_experiment` called too late, or not at all before training | Move it before `.fit()` |
| Parameters are missing entirely | `autolog()` enabled after `.fit()`, or the estimator isn't supported | Enable `mlflow.sklearn.autolog()` before `.fit()`, rerun |
| No run is created at all | Training failed before reaching the instrumented `.fit()` call | Check the traceback; confirm `.fit()` actually executes |
| `curl` fails to connect | Tracking server unreachable | Check the server process/port/URI (Day 20) |
| `200 OK` but no run visible | Reachability ≠ successful tracking | Check the experiment/run directly in the UI |
| `MLmodel` missing from `list_artifacts()` | Model may be a separately-tracked logged model in MLflow 3 | Use `search_logged_models(experiment_ids=[...])` |
| `search_logged_models()` raises a `TypeError` | API requires `experiment_ids` explicitly | Supply it as a list, e.g. `["1"]` |
| Logged model found, but files not visible | Haven't listed its specific artifact location yet | `mlflow.artifacts.list_artifacts(artifact_uri=model.artifact_location, ...)` |
| `penalty` shows as `deprecated` | Installed scikit-learn version reports this value for the estimator | Not a failure signal by itself; check other evidence |
| Pickle security warning appears | About deserializing untrusted pickle files later, not about this run | Informational — treat model files as trusted only if their origin is trusted |
| `FutureWarning` appears | A dependency flagging an upcoming API change | Informational — confirm the run still completed |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.11) out
      loud from "`set_tracking_uri`" to "verified via
      `search_logged_models`."
- [ ] Explain, in one sentence, why `mlflow.sklearn.autolog()` must be
      called *before* `.fit()`, not after.
- [ ] Explain why the UI showed 14 parameters when the script only
      explicitly supplied 3.
- [ ] Explain why `list_artifacts(run_id)` not showing `MLmodel` isn't
      proof that model logging failed in MLflow 3.
- [ ] Explain why `curl -I http://localhost:5000` returning `200 OK`
      doesn't prove autologging captured anything.
- [ ] Explain the difference between a warning and a failure, using
      this lab's pickle warning as the example.
- [ ] From memory, create a second experiment (`autolog-practice`),
      fit a *different* scikit-learn estimator with autologging
      enabled, and independently verify all five layers from §2.9 —
      without copying any command from this file.

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
