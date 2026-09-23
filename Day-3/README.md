# Day 3 — Fix a Broken `uv` Lockfile Specification

A KodeKloud "100 MLOps" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive it yourself*.

---

## 1. Scenario

The xFusionCorp ML team uses `uv` and lockfiles to keep Python
dependencies consistent across machines. A teammate submitted a
`requirements.in` that doesn't match the team's standards.

Broken spec found at `/root/code/fraud-detection/requirements.in`:

```text
# Fraud detection project dependencies
sklearn
mlflow>=100.0
numpy
```

Required end state:
- `requirements.in` lists **exactly** these four top-level packages:
  `scikit-learn`, `mlflow`, `pandas`, `numpy` — any version constraint `uv`
  can actually satisfy against PyPI (bare names are fine; `uv` pins exact
  versions when compiling).
- A pinned `requirements.txt` is compiled from the corrected spec, with
  all four top-level packages pinned via `==`, plus every transitive
  dependency `uv` resolved.

---

## 2. Reasoning model — how to *derive* the fix, not memorize it

### 2.1 The system, mapped

```text
Project dependency intent            Resolved, reproducible environment
        │                                        │
requirements.in    ──────── uv resolver ────────►  requirements.txt
  (what we want)                                  (exactly what gets installed)
```

Two files, two different jobs:

```text
requirements.in   → "what does this project directly depend on?"
requirements.txt  → "what exact, resolved version set reproduces that
                      intent identically on any machine?"
```

Almost every dependency-management task in Python (and the equivalent in
other ecosystems — `package.json`/`package-lock.json`,
`Gemfile`/`Gemfile.lock`, `Pipfile`/`Pipfile.lock`) reduces to this same
two-file shape. Once you see the pattern here, you'll recognize it
everywhere else.

### 2.2 Inspect before touching anything

```bash
cd /root/code/fraud-detection
cat requirements.in
uv --version
```

You can't fix a mismatch you haven't read. Diff what's there against
what's required:

```text
              current              required
line 1        sklearn          →   scikit-learn
line 2        mlflow>=100.0    →   mlflow
line 3        numpy            →   numpy      (unchanged)
(missing)     —                →   pandas
```

Three distinct problems, three distinct root causes — worth separating
rather than treating as "the file is just wrong."

### 2.3 Problem 1 — `sklearn` vs `scikit-learn`: import name ≠ distribution name

```python
import sklearn        # what your code writes
```

```bash
pip install scikit-learn    # what PyPI actually calls the package
```

These are two different namespaces that happen to look similar:

```text
PyPI distribution name   →  scikit-learn   (used in requirements files, pip install)
Python import name        →  sklearn        (used in source code, import statements)
```

`uv`/`pip` resolve packages by **distribution name**, not import name — a
requirements file listing `sklearn` either fails to resolve or resolves
to an unrelated (or nonexistent) package, because the resolver has no way
to know "sklearn" means "scikit-learn." This mismatch isn't unique to
scikit-learn — `PyYAML` imports as `yaml`, `beautifulsoup4` imports as
`bs4`, `Pillow` imports as `PIL`. Whenever a package "isn't found" despite
being clearly installed, checking for an import-name-vs-distribution-name
mismatch is a fast, common diagnosis.

### 2.4 Problem 2 — `mlflow>=100.0`: a constraint, not a comment

```text
mlflow>=100.0
```

read literally by the resolver means:

> "Find me an MLflow release numbered 100.0 or higher."

This isn't decorative — it's a real constraint fed into the solver:

```text
mlflow>=100.0
       │
       ├── does 100.0 exist on PyPI?
       ├── does 100.1 exist?
       ├── does 101.x exist?
       └── ... (as of this writing, none of these exist — MLflow is at 3.x)
```

The task explicitly requires "any version constraint `uv` can satisfy
against PyPI," and explicitly allows bare package names — so the correct
fix is to *remove* the impossible constraint rather than try to guess a
"real" one:

```text
mlflow>=100.0   →   mlflow
```

This changes the request from *"give me version ≥100.0 specifically"* to
*"give me whatever compatible version resolves cleanly"* — letting the
resolver do the version selection, which is exactly what it's for.

### 2.5 Problem 3 — `pandas` is simply missing

The task states the four required top-level packages explicitly:
`scikit-learn`, `mlflow`, `pandas`, `numpy`. The broken file only had
three. This isn't something to infer from scikit-learn's or MLflow's own
dependency trees (both happen to depend on `numpy`, and MLflow happens to
depend on `pandas` too — see §2.8) — it's a required *direct* dependency
because the project's own code needs it, independent of what any other
package transitively pulls in.

### 2.6 Corrected `requirements.in`

```text
# Fraud detection project dependencies
scikit-learn
mlflow
pandas
numpy
```

Notice there are no version numbers here at all — that's intentional, not
an oversight. Bare names say "I directly need these four packages"; the
*exact* versions are the resolver's job, recorded afterward in
`requirements.txt` (§2.7).

### 2.7 `uv pip compile` — what the command actually does

```bash
uv pip compile requirements.in -o requirements.txt
```

```text
uv
 └── pip
      └── compile
           ├── input:  requirements.in   (direct dependency intent)
           └── output: requirements.txt  (fully resolved, pinned graph)
```

You're telling `uv`: *"Read my direct dependencies, resolve the entire
dependency graph including everything they pull in transitively, pick one
mutually-compatible set of versions, and write the exact pins to this
output file."*

### 2.8 Why 4 requested packages produced 88 pinned lines

```text
requirements.in (4 direct/top-level packages)
       │
       ▼
scikit-learn ──┬── numpy
               ├── scipy
               └── joblib

pandas ─────────── numpy

mlflow ─────┬── flask
            ├── sqlalchemy
            ├── pydantic / fastapi / uvicorn
            └── ... (dozens more)
       │
       ▼
   88 resolved packages total
```

Nothing in `requirements.in` mentioned `flask`, `sqlalchemy`, `joblib`, or
`scipy` — none of those are *direct* dependencies of this project. They're
**transitive dependencies**: things scikit-learn, mlflow, and pandas need
internally in order to function. `requirements.txt` has to pin all of
them too, because "reproducible environment" means every package that
ends up installed, not just the ones the project's own code imports.

### 2.9 Dependency resolution — what the solver is actually doing

Conceptually, if package A needs `numpy>=2.0` and package B needs
`numpy<3.0`, the resolver looks for the intersection:

```text
numpy >= 2.0   ∩   numpy < 3.0   =   2.0 <= numpy < 3.0
```

and picks a concrete version inside that range (here, `numpy==2.5.3`). If
two packages' constraints have *no* overlap — e.g. one needs `numpy>=3.0`
and another needs `numpy<3.0` — the intersection is empty, and a correct
resolver **fails loudly** rather than silently installing something
broken. This is the actual value a lockfile-based tool provides beyond
"downloads packages": it proves, before anything is installed, that a
mutually consistent set of versions exists at all.

### 2.10 `requirements.txt` is not "`requirements.in` plus `==`"

This is the misconception the lab is really testing for. Compare:

```text
requirements.in            requirements.txt (excerpt)
────────────────           ──────────────────────────
scikit-learn                aiohappyeyeballs==2.7.1   # via aiohttp
mlflow                      ...
pandas                      mlflow==3.16.1            # via -r requirements.in
numpy                       ...
                             numpy==2.5.3              # via -r requirements.in, +7 others
                             ...
                             scikit-learn==1.9.1       # via -r requirements.in, mlflow, skops
                             ... (84 more lines)
```

The `# via ...` comments `uv` writes are worth reading closely — they show
*why* each package is present: either directly requested
(`-r requirements.in`) or pulled in by something else. This traceability
is exactly what makes a lockfile auditable: for any installed package, you
can answer "why is this here?"

### 2.11 The compressed reasoning chain

```text
Requirement (4 correct top-level deps, pinned lockfile with transitives)
   → Inspect current requirements.in                     → found 3 problems
   → Diagnose: sklearn is an import name, not a PyPI name → scikit-learn
   → Diagnose: mlflow>=100.0 is an unsatisfiable constraint → mlflow (bare)
   → Diagnose: pandas is a required direct dep, missing   → add pandas
   → Rewrite requirements.in with corrected bare names
   → uv pip compile requirements.in -o requirements.txt
   → Resolver builds full dependency graph (4 direct → 88 resolved)
   → requirements.txt written with every package pinned via ==
   → Verify the 4 required top-level packages are present and pinned
```

---

## 3. Concepts (reference)

### 3.1 `requirements.in` vs `requirements.txt`
`requirements.in` is a human-authored statement of *direct* dependency
intent — what the project's own code imports. `requirements.txt` is a
tool-generated, fully resolved snapshot of the *entire* dependency graph
(direct + transitive), each pinned to one exact version. The first
changes rarely and by hand; the second is regenerated by the resolver
whenever the first changes.

### 3.2 Distribution name vs. import name
The name used in `pip install <name>` / `requirements.in` (the PyPI
project name) is not guaranteed to match the name used in
`import <name>` in source code. `scikit-learn`/`sklearn`,
`PyYAML`/`yaml`, `beautifulsoup4`/`bs4`, and `Pillow`/`PIL` are the
classic examples. Package managers resolve by distribution name; Python's
import system resolves by import name — two independent namespaces.

### 3.3 Version specifiers as solver constraints
`>=`, `<=`, `==`, `~=`, `!=` in a requirements file are not comments or
suggestions — they're constraints fed directly into the dependency
resolver's search space. An impossible or absurdly high constraint (like
`mlflow>=100.0` against a package that has never released past `3.x`)
simply has no satisfying version, and resolution fails or the constraint
must be relaxed.

### 3.4 Direct vs. transitive dependencies
A **direct** dependency is one your project's own code imports and that
you declare explicitly. A **transitive** dependency is one pulled in
because *a direct dependency* needs it internally (e.g. `mlflow` needing
`flask` to serve its tracking UI). A full lockfile must pin both — an
environment is only reproducible if every installed package, not just the
ones you explicitly asked for, is pinned.

### 3.5 Lockfile pinning (`==`)
A bare `pandas` in a spec file says "any version satisfying no particular
constraint." `pandas==3.0.6` in a lockfile says "exactly this version,
every time, on every machine." Pinning eliminates "works on my machine"
drift caused by two installs resolving the same loose spec to two
different actual versions at different points in time.

### 3.6 Why reproducibility matters specifically for ML
An ML pipeline's behavior can depend on the exact versions of `numpy`,
`pandas`, and `scikit-learn` — numerical results, serialization formats,
and preprocessing behavior can all shift subtly across versions. Pinning
the full environment (not just top-level packages) is what lets a
training run, a CI pipeline, and a production inference server all use
provably identical dependency code.

---

## 4. Runbook

### 4.1 Inspect current state
```bash
cd /root/code/fraud-detection

cat requirements.in
```
```text
# Fraud detection project dependencies
sklearn
mlflow>=100.0
numpy
```

```bash
uv --version
```
```text
uv 0.12.18 (x86_64-unknown-linux-gnu)
```

### 4.2 Diagnose the three problems (§2.3–§2.5)
```text
sklearn         → wrong: PyPI distribution name is scikit-learn
mlflow>=100.0   → wrong: unsatisfiable constraint, no such version exists
pandas          → missing: required direct dependency
```

### 4.3 Correct the specification
```bash
cat > requirements.in <<'EOF'
scikit-learn
mlflow
pandas
numpy
EOF
```

Verify it landed as written:
```bash
cat requirements.in
```
```text
# Fraud detection project dependencies
scikit-learn
mlflow
pandas
numpy
```

### 4.4 Compile the pinned lockfile
```bash
uv pip compile requirements.in -o requirements.txt
```
```text
Resolved 88 packages in 551ms
```

### 4.5 Verify — the four required top-level packages are pinned
```bash
awk -F'==' '$1=="scikit-learn" || $1=="mlflow" || $1=="pandas" || $1=="numpy" {print}' requirements.txt
```
```text
mlflow==3.16.1
numpy==2.5.3
pandas==3.0.6
scikit-learn==1.9.1
```

### 4.6 Spot-check traceability of a transitive dependency
```bash
grep -A2 "^scipy==" requirements.txt
```
```text
scipy==1.18.1
    # via
    #   mlflow
    #   scikit-learn
    #   skops
```
Confirms `scipy` is present because scikit-learn and mlflow need it, not
because it was ever listed in `requirements.in` — exactly the direct vs.
transitive distinction from §3.4.

### 4.7 Submit
With `requirements.in` holding exactly the four required top-level
packages and `requirements.txt` fully pinned and resolved, click **Check**
in the KodeKloud lab UI.

---

## 5. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `error: Distribution not found` or resolution fails on `sklearn` | Used the import name instead of the PyPI distribution name | Use `scikit-learn` in `requirements.in`, not `sklearn` |
| Resolver reports no version satisfies `mlflow>=100.0` | Constraint sets a lower bound higher than any real release | Remove the constraint; use a bare `mlflow` or a constraint you've confirmed exists on PyPI |
| Lab check fails: "missing required top-level package" | `pandas` (or another required package) left out of `requirements.in` | Re-check the task's required package list against the actual file contents |
| `requirements.txt` missing transitive deps / looks too short | Compiled from an incomplete or still-broken `requirements.in` | Fix `requirements.in` first, then re-run `uv pip compile` — don't hand-edit `requirements.txt` |
| Unsure why a package appears in `requirements.txt` | Forgot it's a transitive dependency, not something you added | Read the `# via ...` comment `uv` writes under each pinned line (§2.10) |

---

## 6. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.11) out loud
      from "here's a broken `requirements.in`" to "here's a verified,
      pinned `requirements.txt`."
- [ ] Explain, in one sentence, the difference between a package's PyPI
      distribution name and its Python import name, with an example other
      than scikit-learn.
- [ ] Explain why `mlflow>=100.0` is a *constraint* fed to a solver, not
      just descriptive text, and why it made resolution impossible.
- [ ] Explain the difference between a direct dependency and a transitive
      dependency, using this lab's own dependency graph as the example.
- [ ] Explain why `requirements.txt` should never be hand-edited to "add"
      a missing package — what should you edit instead, and why?
- [ ] Break `requirements.in` again (revert to the original broken
      version) and fix it from memory, then re-run `uv pip compile` and
      re-verify the four pinned top-level packages.

---

## 7. Document template for future lab days

Use this skeleton for `day-N/README.md`:

```markdown
# Day N — <task title>

## 1. Scenario
<what was asked, what was broken/required, constraints>

## 2. Reasoning model — how to derive the fix/commands
<walk the requirement down step by step, in the order you'd actually
discover it: inspect current state → identify the gap vs. required state →
diagnose each root cause individually → fix → re-run → verify the actual
success state. End with a compressed arrow-chain version.>

## 3. Concepts (reference)
<one subsection per concept the task exercises — explain WHY, not just WHAT>

## 4. Runbook
<copy-pasteable commands in the order run, with real intermediate
output/results inline where they mattered to the diagnosis>

## 5. Troubleshooting
<symptom / cause / fix table>

## 6. Do it yourself (checklist)
<"walk the reasoning chain out loud" + comprehension questions + "break it
again and fix it from memory" prompt>
```
