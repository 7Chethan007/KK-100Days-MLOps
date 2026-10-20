# Day 26 — Log a Model with a Signature and Validate Inputs

A KodeKloud "100 MLOps" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive it yourself*.

Your fraud-detection model works perfectly in a notebook. Then a
transaction arrives with one feature missing, and the model either
crashes somewhere deep inside, or — worse — silently predicts on
garbage. Day 24 (autologging) captured *what* the model's parameters
were. Day 25 (registry) gave it a *name and a version*. Neither one
gives the model an enforceable *input contract* — that's this lab's
entire job, and it's the piece that turns "a model that predicts"
into "a model safe enough to put behind an API."

---

## 1. Scenario

Complete `/root/code/log_with_signature.py` so that a
`DummyClassifier` trained on `amount`/`num_txn` is logged into the
`fraud-signature` experiment with an inferred signature and an input
example, then prove that MLflow's `pyfunc` interface accepts valid
input and *rejects* input missing a required feature.

| Requirement | Expected |
|---|---|
| Experiment | `fraud-signature` |
| Signature | `amount: double (required)`, `num_txn: long (required)` |
| Output | `Tensor('int64', (-1,))` |
| Run | `FINISHED` |
| Valid input `{amount, num_txn}` | prediction succeeds |
| Invalid input `{amount}` only | prediction **rejected** |

```text
X (amount, num_txn) ──┐
                       ├──▶ infer_signature(X, preds) ──▶ signature
model.fit(X, y)       │
                       └──▶ preds
                                │
                                ▼
mlflow.sklearn.log_model(model, name="model",
                          signature=signature, input_example=X)
                                │
                                ▼
                 fraud-signature experiment, run FINISHED
                                │
                 mlflow.pyfunc.load_model(...)
                      ├── valid {amount, num_txn}   → prediction
                      └── invalid {amount only}     → REJECTED
```

---

## 2. Reasoning model — how to *derive* the code, not memorize it

### 2.1 A signature is a *contract*, not a side effect of training

```text
model.fit(X, y)              → teaches the model a prediction function
infer_signature(X, preds)    → separately DESCRIBES what that function expects/returns
```

Nothing about calling `.fit()` produces a schema — `infer_signature`
is an entirely separate, deliberate step that looks at the *shape* of
the training inputs and outputs and encodes it as metadata. A model
can be perfectly well-trained and have zero signature if this step is
skipped — the same "identity vs. authorization" split that's run
through this whole series (Day 20's IAM role, Day 25's registered
model vs. version) shows up here as "the model's learned behavior vs.
the model's declared interface."

### 2.2 The signature describes *structure*, not correctness — of the schema, the data, or the model

```text
'amount': double (required), 'num_txn': long (required)
```

This tells a caller exactly two things: which columns must be
present, and what type each must be. It says *nothing* about whether
`amount` should be non-negative, whether 1,000,000 training examples
back this model, or whether it's a good fraud detector at all —
those are entirely different kinds of correctness, each needing its
own separate validation layer. Conflating "has a well-formed
signature" with "is production-ready" is exactly the trap this lab
is built to expose.

### 2.3 Input example and signature are two different, complementary artifacts — one isn't the other

```text
signature:      THE RULE   — "amount and num_txn, both required, these types"
input_example:  ONE SAMPLE — {"amount": 250.0, "num_txn": 6}
```

A sample containing exactly two columns doesn't, by itself, declare
that both are mandatory — that declaration is the signature's job
specifically. The input example exists for a different reason:
giving tooling (and humans) a concrete instance of "here's what a
valid call looks like," useful for serving setup and documentation,
but carrying no enforcement power of its own.

### 2.4 Checking the source code and checking the logged metadata are two separate verification layers

```text
grep -n 'infer_signature\|log_model' log_with_signature.py   → confirms INTENT (what the code says it does)
mlflow.models.get_model_info(model_uri).signature             → confirms RESULT (what actually got saved)
```

Reading the script tells you what the author *meant* to log; reading
the model's saved metadata via `get_model_info` tells you what MLflow
*actually* recorded. A typo, a wrong variable, or a version
difference could make these diverge — this is the same
"inspect-before-acting, then verify-after-acting" two-step as every
CLI/config lab in this series, just applied to Python/MLflow instead
of AWS or Nginx.

### 2.5 A `FINISHED` run proves the *tracking* succeeded — not that the schema is correct

```text
run_id: 0fb28d3e...   status: FINISHED
```

"The script ran to completion and MLflow recorded a run" and "the
logged signature correctly matches what I expect" are two entirely
separate claims. This is the Day 20 AWS/Day 24 MLOps "an HTTP 200 / a
successful process exit doesn't prove the specific requirement was
met" pattern recurring yet again — a `FINISHED` status is necessary,
not sufficient.

### 2.6 Signature *validation* only happens at MLflow's `pyfunc` boundary — not by calling the raw scikit-learn object directly

```text
model.predict(invalid)                                  ← via mlflow.pyfunc.load_model(...) → REJECTED
raw_sklearn_estimator.predict(invalid_numpy_array)        ← bypasses MLflow entirely → may just crash or misbehave
```

The signature is stored as metadata *alongside* the model artifact —
it's MLflow's `pyfunc` loading/serving layer that actually reads that
metadata and enforces it on every call. Calling the underlying
scikit-learn estimator directly (skipping `mlflow.pyfunc.load_model`)
bypasses the contract entirely, because nothing in plain scikit-learn
knows the signature exists. This is *the* critical distinction the
lab's §6.E troubleshooting entry is built around: "invalid input is
accepted" almost always means the wrong loading path was used, not
that validation itself is broken.

### 2.7 Testing *both* a valid and an invalid request is the only way to prove the contract is actually enforced, not merely declared

```text
Test A (valid, both columns)     → should SUCCEED
Test B (invalid, missing num_txn) → should be REJECTED
```

Running only Test A would leave open the possibility that the model
accepts *literally anything*, signature or no signature — success on
valid input alone proves nothing about rejection behavior. The
decisive proof that the contract is enforced, not just documented,
is specifically that malformed input *fails* when it's supposed to.

### 2.8 The compressed reasoning chain

```text
Requirement: a model with an enforced input contract
   → inspect the script FIRST — confirm infer_signature(X, preds) and log_model(...) are present (§2.4)
   → infer_signature(X, preds) — a SEPARATE step from .fit(), describes structure only (§2.1, §2.2)
   → log_model(model, name="model", signature=signature, input_example=X) — attach the contract (§2.3)
   → run the script → confirm the run status is FINISHED — necessary, not sufficient (§2.5)
   → get_model_info(model_uri).signature — verify the SAVED metadata, not just the source code (§2.4)
   → mlflow.pyfunc.load_model(model_uri) — the ONLY interface that enforces the signature (§2.6)
   → Test A: valid input → succeeds
   → Test B: invalid (missing num_txn) → REJECTED — the decisive proof of enforcement (§2.7)
```

---

## 3. Concepts (reference)

### 3.1 Model signature
A machine-readable description of a model's expected input schema
(column names, types, required/optional) and output schema, stored
alongside the logged model artifact.

### 3.2 `infer_signature(X, preds)`
Derives a signature by inspecting the structure of representative
input data and the predictions produced from it — a separate,
explicit step from training the model itself.

### 3.3 Input example vs. signature
An input example is one concrete sample of valid input, useful for
documentation and certain tooling. The signature is the actual
enforced schema rule. Neither substitutes for the other.

### 3.4 `mlflow.pyfunc.load_model()` as the enforcement boundary
Loading a model through MLflow's standardized `pyfunc` interface is
what actually applies signature validation on every prediction call —
calling the raw underlying framework object bypasses this entirely.

### 3.5 Contract correctness vs. model/data correctness
A signature validates *structure* (right columns, right types). It
says nothing about business-rule validity (e.g. non-negative amounts)
or model quality (e.g. real predictive accuracy) — those require
separate, additional validation layers.

### 3.6 Verifying intent vs. verifying result
Reading source code confirms what was *intended*; reading the logged
model's actual metadata (via `get_model_info`) confirms what was
*recorded* — these can diverge, and only the second is decisive proof.

---

## 4. Runbook

### 4.1 Inspect the prepared script
```bash
cd /root/code
ls -l log_with_signature.py
grep -n -A 8 -B 5 'infer_signature\|log_model' log_with_signature.py
```
Confirms `infer_signature`, `log_model(..., signature=..., input_example=...)`,
tracking URI `http://localhost:5000`, experiment `fraud-signature`.

### 4.2 Key code — signature inference
```python
signature = infer_signature(X, preds)
```
Expected inferred schema:
```text
inputs:
  ['amount': double (required), 'num_txn': long (required)]
outputs:
  [Tensor('int64', (-1,))]
params:
  None
```

### 4.3 Key code — logging with the contract attached
```python
with mlflow.start_run():
    mlflow.sklearn.log_model(
        model,
        name="model",
        signature=signature,
        input_example=X,
    )
```

### 4.4 Execute the script
```bash
python3 /root/code/log_with_signature.py
```
```text
model logged to the fraud-signature experiment
```

### 4.5 Verify the run exists and is `FINISHED`
```python
import mlflow
mlflow.set_tracking_uri("http://localhost:5000")
experiment = mlflow.get_experiment_by_name("fraud-signature")
runs = mlflow.search_runs(
    experiment_ids=[experiment.experiment_id],
    order_by=["start_time DESC"], max_results=1,
)
print(runs[["run_id", "status"]].to_string(index=False))
```
```text
run_id                            status
0fb28d3eafaf41608d05a8dd1fc69346  FINISHED
```

### 4.6 Verify the saved signature — the decisive metadata check
```python
run_id = "0fb28d3eafaf41608d05a8dd1fc69346"
model_uri = f"runs:/{run_id}/model"
model_info = mlflow.models.get_model_info(model_uri)
print(model_info.signature)
print(model_info.saved_input_example_info)
```
```text
inputs:
  ['amount': double (required), 'num_txn': long (required)]
outputs:
  [Tensor('int64', (-1,))]
params:
  None

{'artifact_path': 'input_example.json', 'pandas_orient': 'split',
 'serving_input_path': 'serving_input_example.json', 'type': 'dataframe'}
```

### 4.7 Test both valid and invalid input via `pyfunc` — the actual contract enforcement test
```python
import mlflow, pandas as pd
mlflow.set_tracking_uri("http://localhost:5000")
model = mlflow.pyfunc.load_model(f"runs:/{run_id}/model")

valid = pd.DataFrame({"amount": [12.0], "num_txn": [1]})
print("Prediction:", model.predict(valid))

invalid = pd.DataFrame({"amount": [12.0]})
try:
    model.predict(invalid)
    print("FAIL: Invalid input was unexpectedly accepted")
except Exception as exc:
    print("PASS: Invalid input was rejected")
    print(type(exc).__name__, str(exc)[:300])
```
Expected: Test A succeeds, Test B raises and prints `PASS`.

---

## 5. Final state

| Requirement | Evidence | Result |
|---|---|---|
| `infer_signature(X, preds)` used | source inspection | ✅ |
| Model logged with `signature` + `input_example` | source inspection | ✅ |
| Run completed | `status: FINISHED` | ✅ |
| Saved signature correct | `get_model_info(...).signature` | ✅ |
| Valid input `{amount, num_txn}` → prediction succeeds | Test A | ✅ |
| Invalid input `{amount}` only → rejected | Test B | ✅ |

```text
fraud-signature experiment
 └── run (0fb28d3e..., FINISHED)
       └── logged model: "model"
              signature: amount(double,req) + num_txn(long,req) → int64 tensor
              input_example: {"amount": ..., "num_txn": ...}
              │
              loaded via mlflow.pyfunc.load_model(...)
                ├── {amount, num_txn}   → prediction ✅
                └── {amount only}        → rejected  ✅
```

---

## 6. Troubleshooting

| Symptom | Likely cause | Check |
|---|---|---|
| Script fails before logging | Import/syntax error, or tracking server unreachable | Full traceback; `curl -I http://localhost:5000` |
| Run missing or not `FINISHED` | Wrong experiment name/tracking URI, or script errored mid-run | Re-check `set_tracking_uri`/`set_experiment`, inspect script output |
| Signature missing or wrong | `infer_signature` args wrong, or `signature=` not passed to `log_model` | Re-check `infer_signature(X, preds)` and the `log_model(...)` call |
| Valid input fails | Column names/types/structure don't match the saved signature | Compare the DataFrame against `get_model_info(...).signature` exactly |
| Invalid input is *accepted* (the dangerous failure) | Loaded the raw scikit-learn estimator instead of via `mlflow.pyfunc.load_model` | Confirm you're calling the `pyfunc`-loaded model, not the bare sklearn object |
| Confusing downstream errors from bad input | Validation happening too late, deep inside business logic | Validate at the serving boundary (this lab's whole point), not after |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.8) out loud
      from "`infer_signature`" to "Test B rejected via `pyfunc`."
- [ ] Explain, in one sentence, why a signature existing doesn't mean
      the model is accurate or the training data is good.
- [ ] Explain why calling the raw scikit-learn estimator's `.predict()`
      directly would *not* enforce the signature, even though the
      signature was correctly logged.
- [ ] Explain why checking the source code isn't sufficient
      verification — what does `get_model_info` tell you that
      reading the script doesn't?
- [ ] Explain why testing only the valid-input case would leave the
      lab's actual requirement unproven.
- [ ] From memory, log a second model with a *three*-feature
      signature, test it with all three features present, then with
      one missing, and confirm the rejection — without copying any
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
