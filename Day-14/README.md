# Day 14 — DVC Pipelines from First Principles

A KodeKloud "100 MLOps" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive the
`dvc.yaml` yourself, from the scripts alone*.

This is the next step in the DVC arc running through Days 10–13 (init →
add → push → pull). Those labs were all about versioning individual
*files*. Today is about versioning the *process* that produces them —
the data pipeline itself, as a reproducible, dependency-aware graph.

---

## 1. Scenario

The `fraud-detection` project has two existing scripts but no DVC
pipeline wiring them together:

```text
src/data/process_data.py   → reads data/raw/transactions.csv
                              writes data/processed/clean_transactions.csv

src/data/split_data.py     → reads data/processed/clean_transactions.csv
                              writes data/processed/train.csv, test.csv
```

Task: define both as DVC pipeline stages in `dvc.yaml`, run the
pipeline via `dvc repro`, and confirm DVC considers it up to date
(`dvc status`) with the correct dependency graph (`dvc dag`).

---

## 2. Reasoning model — how to *derive* the pipeline, not memorize the YAML

### 2.1 The problem one level up from file versioning

Days 10–13 made individual files (`transactions.csv`) reproducible —
you could always get back the exact bytes a `.dvc` pointer specified.
But a real ML workflow isn't one file, it's a **chain of transformations**:

```text
transactions.csv → process → clean_transactions.csv → split → train.csv + test.csv
```

"How exactly were `train.csv` and `test.csv` generated?" is a question
file-level versioning alone can't answer — you'd need to also know
*which script*, in *what order*, with *what inputs*. A DVC **pipeline**
is the mechanism that captures the process itself, not just its
outputs.

### 2.2 Derive the stage definition from the scripts — don't guess at YAML syntax first

```bash
sed -n '1,220p' src/data/process_data.py
```
```python
df = pd.read_csv("data/raw/transactions.csv")
df = df.drop_duplicates()
df = df.dropna()
df.to_csv("data/processed/clean_transactions.csv", index=False)
```

Reading the actual script tells you everything a stage definition
needs, directly:

```text
what it reads   → data/raw/transactions.csv           (a dependency)
what it IS       → src/data/process_data.py             (also a dependency — §2.4)
what it writes   → data/processed/clean_transactions.csv (an output)
```

Same for `split_data.py` — read it first, derive its deps/outs from
what it actually does, rather than inventing a `dvc.yaml` from
assumptions about what a "typical" pipeline looks like.

### 2.3 A stage is exactly three things: `cmd`, `deps`, `outs`

```yaml
process_data:
  cmd: python3 src/data/process_data.py
  deps:
    - data/raw/transactions.csv
    - src/data/process_data.py
  outs:
    - data/processed/clean_transactions.csv
```

```text
cmd  → WHAT TO RUN       (the recipe)
deps → WHAT IT NEEDS      (inputs — triggers a rerun if any of these change)
outs → WHAT IT PRODUCES   (outputs — what DVC tracks the result of)
```

This three-part shape is the entire mental model for any DVC stage,
regardless of how complex the underlying script is.

### 2.4 The script itself is a dependency — not just the data it reads

```yaml
deps:
  - data/raw/transactions.csv      # the data
  - src/data/process_data.py       # the CODE
```

The common beginner mistake is treating `deps` as "only the data files."
But an ML pipeline is really `output = function(input)`, and the
*function* is the code — changing `df.drop_duplicates()` to
`df.drop_duplicates().dropna()` changes the output even if the raw data
file is byte-for-byte identical. DVC has no way to know the code
changed unless the code file is itself declared as a dependency — this
is exactly why the stage lists the `.py` file alongside the `.csv` file,
not as an afterthought but as an equally load-bearing fact about what
the output actually depends on.

### 2.5 Stage chaining — DVC infers the pipeline order from matching filenames, not from an explicit link

```yaml
process_data:
  outs:
    - data/processed/clean_transactions.csv   # OUTPUT of this stage

split_data:
  deps:
    - data/processed/clean_transactions.csv   # INPUT of this stage — SAME path
```

Nowhere in `dvc.yaml` do you write "`process_data` runs before
`split_data`." DVC derives that ordering automatically because one
stage's `outs` entry is textually identical to another stage's `deps`
entry — this is the entire chaining mechanism, and it's exactly why
getting the file paths to match *exactly* (same relative path, same
filename) between a producing stage's `outs` and a consuming stage's
`deps` is what makes chaining work at all; a typo here silently breaks
the chain without DVC necessarily erroring.

### 2.6 This is a DAG — and why "acyclic" specifically matters

```text
process_data → split_data   (directed: order matters, not symmetric)
```

A pipeline can never have `A → B → A` — DVC's model assumes you can
always determine a valid execution order by following dependency
arrows forward; a cycle would make "what runs first" undefined. This is
the same directed-acyclic-graph structure underlying build systems
(Make — Day 5's lab), task schedulers, and most dependency-resolution
systems generally, not something unique to DVC.

### 2.7 `dvc dag` — ask DVC what it understood, before running anything

```bash
dvc dag
```
```text
+--------------+
| process_data |
+--------------+
       *
       *
 +------------+
 | split_data |
 +------------+
```

This is a genuinely useful verification step *before* `dvc repro` —
it confirms DVC inferred the same stage ordering you intended, purely
from the `dvc.yaml` you wrote, before spending time actually executing
anything. If the DAG looks wrong (missing an edge, or stages appearing
unconnected), the `deps`/`outs` paths don't actually match somewhere
(§2.5) — fix that before running `dvc repro`, not after.

### 2.8 `dvc repro` — execute in dependency order, not file-list order

```bash
dvc repro
```
```text
Running stage 'process_data':
> python3 src/data/process_data.py
Processed 15 rows

Running stage 'split_data':
> python3 src/data/split_data.py
Train: 12 rows, Test: 3 rows
```

`split_data` cannot run before `process_data` because its own
dependency (`clean_transactions.csv`) doesn't exist until
`process_data` produces it — the same "you can't run tests before the
binary compiles" ordering constraint from any build system. DVC
resolves this automatically from the DAG; you never manually sequence
the two `python3` calls yourself.

### 2.9 `dvc.lock` — the recorded state of what actually ran, distinct from the intended definition

```text
dvc.yaml   → WHAT THE PIPELINE SHOULD LOOK LIKE   (the recipe, hand-authored)
dvc.lock   → WHAT ACTUALLY RAN AND PRODUCED        (recorded automatically by dvc repro)
```

This is the same `.dvc`-pointer-style content-hashing idea from Day 11,
now applied at the pipeline level instead of a single file — `dvc.lock`
records the hashes of each stage's actual deps/outs at the moment it
successfully ran, which is what lets a later `dvc status`/`dvc repro`
determine "has anything relevant changed since last time," without
re-reading every file's content from scratch each time.

### 2.10 How DVC decides a stage doesn't need to rerun

```text
dvc repro (second run, nothing changed)
        │
        ▼
Compare current file hashes against dvc.lock's recorded hashes
        │
        ▼
No differences → stage is SKIPPED, not re-executed
```

This is the actual payoff of the lock file + content hashing combo —
`dvc repro` on an unchanged project doesn't blindly re-run every script;
it specifically checks whether each stage's recorded deps/outs still
match reality, and only re-executes stages where something genuinely
changed. At scale (a 10-stage pipeline, one changed input), this is the
difference between re-running everything and re-running only what's
actually affected downstream.

### 2.11 `dvc status` vs. `dvc repro` — inspect vs. execute

```text
dvc status → "is anything stale?"            (read-only, no side effects)
dvc repro  → "reproduce whatever IS stale"    (actually runs commands)
```

Running `dvc status` before `dvc repro` lets you see what *would*
happen without committing to actually re-running scripts — useful for
confirming your mental model of what changed matches DVC's, before
triggering potentially expensive recomputation.

### 2.12 Running the scripts manually instead of through `dvc repro` defeats the entire point

```bash
python3 src/data/process_data.py    # works, but DVC never records it happened
python3 src/data/split_data.py       # same — no dvc.lock update, no staleness tracking
```

This would produce the identical output files, but DVC would have no
record of *when* or *under what input state* they were produced — the
whole reproducibility guarantee this lab is building depends on routing
every pipeline execution through `dvc repro`, not around it.

### 2.13 The compressed reasoning chain

```text
Requirement (define + run a 2-stage DVC pipeline, verify reproducible)
   → Read process_data.py                    → derive its deps (data + code) and outs
   → Read split_data.py                       → derive its deps (clean csv + code) and outs (train/test)
   → Write dvc.yaml with cmd/deps/outs per stage, using python3 explicitly
   → Ensure split_data's dep path EXACTLY matches process_data's out path (chaining, §2.5)
   → dvc dag                                    → confirm DVC inferred process_data → split_data
   → dvc repro                                    → runs IN DAG ORDER; writes dvc.lock
   → Inspect data/processed/ for clean_transactions.csv, train.csv, test.csv
   → dvc status                                    → "up to date" (lock matches current state)
   → (second dvc repro with nothing changed)        → stages correctly SKIPPED, not re-run
```

---

## 3. Concepts (reference)

### 3.1 A DVC stage's three parts
`cmd` (what to run), `deps` (what triggers a rerun if changed), `outs`
(what the stage is responsible for producing). Every stage in every
`dvc.yaml` reduces to these three.

### 3.2 Code as a dependency, not just data
An ML pipeline's output is a function of both its input data and the
code processing it — both must be declared as `deps` for DVC's
staleness detection to be meaningful. Omitting the code file from
`deps` means a code change would silently go undetected by `dvc
status`.

### 3.3 Implicit stage chaining via matching paths
DVC never needs an explicit "stage A runs before stage B" declaration —
it derives the entire execution order from one stage's `outs` matching
another stage's `deps`, textually. This makes `dvc.yaml` purely
declarative (describe what each stage needs and produces) rather than
imperative (describe the order to run things in).

### 3.4 DAG (Directed Acyclic Graph)
Directed: dependency order is one-way. Acyclic: no stage can
(transitively) depend on itself. The same structural pattern underlies
Makefiles (Day 5), build systems generally, and task schedulers — DVC's
pipeline model isn't a novel invention, it's this same well-established
structure applied to data processing specifically.

### 3.5 `dvc.yaml` vs. `dvc.lock`
`dvc.yaml` is hand-authored intent — what the pipeline *should* do.
`dvc.lock` is machine-generated record — what *actually* ran and what
hashes resulted, used by DVC to detect staleness on future runs. Commit
both to Git; they serve different, complementary purposes.

### 3.6 `dvc dag` / `dvc repro` / `dvc status`
- `dvc dag` — **see** the inferred dependency graph (read-only,
  diagnostic).
- `dvc repro` — **run** whatever stages are stale, in dependency order.
- `dvc status` — **check** whether anything is stale, without running
  anything.

### 3.7 Why unnecessary re-execution is avoided
Content-hash comparison against `dvc.lock` (the same hashing principle
from Day 11's `.dvc` pointers) lets DVC skip stages whose deps/outs
haven't actually changed — critical for pipelines with expensive stages
(training runs, large data processing) where blind re-execution would
waste significant time/compute on unaffected work.

---

## 4. Runbook

### 4.1 Confirm no pipeline exists yet
```bash
cd /root/code/fraud-detection
ls dvc.yaml dvc.lock 2>&1
```
```text
ls: cannot access 'dvc.yaml': No such file or directory
ls: cannot access 'dvc.lock': No such file or directory
```

### 4.2 Read the processing script to derive stage 1
```bash
sed -n '1,220p' src/data/process_data.py
```
```python
df = pd.read_csv("data/raw/transactions.csv")
df = df.drop_duplicates()
df = df.dropna()
df.to_csv("data/processed/clean_transactions.csv", index=False)
```

### 4.3 Read the splitting script to derive stage 2
```bash
sed -n '1,220p' src/data/split_data.py
```
```python
df = pd.read_csv("data/processed/clean_transactions.csv")
...
train.to_csv("data/processed/train.csv", index=False)
test.to_csv("data/processed/test.csv", index=False)
```

### 4.4 Write dvc.yaml from what the scripts actually read/write
```bash
cat > dvc.yaml <<'EOF'
stages:
  process_data:
    cmd: python3 src/data/process_data.py
    deps:
      - data/raw/transactions.csv
      - src/data/process_data.py
    outs:
      - data/processed/clean_transactions.csv

  split_data:
    cmd: python3 src/data/split_data.py
    deps:
      - data/processed/clean_transactions.csv
      - src/data/split_data.py
    outs:
      - data/processed/train.csv
      - data/processed/test.csv
EOF
```

### 4.5 Confirm DVC infers the correct dependency graph
```bash
dvc dag
```
```text
+--------------+
| process_data |
+--------------+
       *
       *
 +------------+
 | split_data |
 +------------+
```

### 4.6 Run the pipeline
```bash
dvc repro
```
```text
Running stage 'process_data':
> python3 src/data/process_data.py
Processed 15 rows

Running stage 'split_data':
> python3 src/data/split_data.py
Train: 12 rows, Test: 3 rows
```

### 4.7 Confirm the outputs and lock file
```bash
ls data/processed/
```
```text
clean_transactions.csv  train.csv  test.csv
```
```bash
ls dvc.lock
```

### 4.8 Confirm the pipeline reports up to date
```bash
dvc status
```
```text
Data and pipelines are up to date.
```

### 4.9 Confirm re-running does nothing unnecessary (second repro)
```bash
dvc repro
```
```text
Stage 'process_data' didn't change, skipping
Stage 'split_data' didn't change, skipping
```

### 4.10 Submit
With `dvc dag` showing the correct graph, `dvc repro` having produced
all three output files, `dvc.lock` present, and `dvc status` reporting
up to date, click **Check** in the KodeKloud lab UI.

---

## 5. Final state

| Requirement | Result |
|---|---|
| `dvc.yaml` defines `process_data` and `split_data` stages | ✅ |
| Each stage uses `python3` explicitly | ✅ |
| Each stage's `deps` includes both data AND code | ✅ |
| `split_data`'s dep path matches `process_data`'s out path exactly | ✅ |
| `dvc dag` shows `process_data → split_data` | ✅ |
| `dvc repro` runs both stages in order, produces all outputs | ✅ |
| `dvc.lock` created | ✅ |
| `dvc status` reports up to date | ✅ |

```text
dvc.yaml
   │
   ├── process_data: cmd + deps(data, code) + outs(clean_transactions.csv)
   │                                                 │
   └── split_data:    cmd + deps(──────────────────┘, code) + outs(train.csv, test.csv)
             │
             ▼
        dvc repro
             │
             ▼
  data/processed/{clean_transactions,train,test}.csv
             │
             ▼
        dvc.lock written
             │
             ▼
     dvc status → up to date
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `dvc dag` shows two disconnected stages instead of a chain | A path in one stage's `outs` doesn't textually match the other stage's `deps` (typo, different relative path) | Compare the two path strings character-for-character; DVC chains purely on exact path match |
| `dvc status` doesn't detect a code change | The script file was never added to that stage's `deps` | Add the `.py` file explicitly to `deps` — code is a dependency, not just data (§2.4) |
| `dvc repro` runs `split_data` before `process_data`, or fails looking for an input | `dvc.yaml`'s `deps`/`outs` don't correctly express the real chain | Re-derive from the scripts' actual read/write paths, not from assumed stage order |
| Used `python` instead of `python3` in `cmd` | Task-specific requirement not followed | Set `cmd: python3 <script>` explicitly |
| Ran the scripts manually instead of via `dvc repro` | Produces correct files but no `dvc.lock` update, no staleness tracking | Always route pipeline execution through `dvc repro`, never run the scripts directly when DVC is managing the pipeline |
| Second `dvc repro` re-runs everything even though nothing changed | `dvc.lock` missing, deleted, or a dependency's hash is unexpectedly different each run (e.g. a non-deterministic script) | Confirm `dvc.lock` exists and is committed; investigate why a dependency's content differs between runs if it shouldn't |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.13) out loud
      from "two scripts, no pipeline" to "dvc status: up to date,
      re-running does nothing unnecessary."
- [ ] Explain, in one sentence, why a stage's code file belongs in
      `deps` alongside its data file.
- [ ] Explain how DVC determines that `split_data` must run after
      `process_data`, without that order being written anywhere
      explicitly.
- [ ] Explain the difference between `dvc.yaml` and `dvc.lock` — which
      one would you hand-edit, and which one is machine-generated?
- [ ] Explain the difference between `dvc dag`, `dvc repro`, and `dvc
      status` — which one actually executes anything?
- [ ] Explain why running `python3 process_data.py` directly, instead
      of `dvc repro`, undermines the point of defining a pipeline at
      all.
- [ ] Add a third stage from memory (e.g. a trivial script producing a
      summary file from `train.csv`), wire its `deps` to match
      `split_data`'s `outs` exactly, and confirm `dvc dag` shows the
      extended three-stage chain before running `dvc repro`.

---

## 8. Document template for future lab days

Use this skeleton for `day-N/README.md`:

```markdown
# Day N — <task title>

## 1. Scenario
<what was asked, what was broken/required, constraints>

## 2. Reasoning model — how to derive the fix/commands
<walk the requirement down step by step, in the order you'd actually
discover it: read the actual scripts/existing artifacts to derive
inputs/outputs → express that as the tool's native structure (stages,
rules, etc.) → verify the tool's OWN understanding (dag/plan/dry-run)
before executing → execute → verify the recorded/lock state → confirm
re-running does the minimal necessary work. End with a compressed
arrow-chain version.>

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
