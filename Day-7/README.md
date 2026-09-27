# Day 7 — Test and Package the Fraud-Detection Module

A KodeKloud "100 MLOps" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive the fix
yourself*.

---

## 1. Scenario

The xFusionCorp deployment team needs the `fraud-detection` module:

1. Validated with unit tests.
2. Packaged as a proper, installable Python distribution.
3. Built into a compliant wheel.

Project at `/root/code/fraud-detection/`, `src/` layout:

```text
src/fraud_detection/__init__.py
src/fraud_detection/predict.py
```

```python
def predict(data: Iterable[Iterable[float]]) -> List[int]:
    """Return a fraud label (0 or 1) for each input feature row.
    Flags any transaction whose first feature value exceeds 100.
    """
    return [1 if row[0] > 100 else 0 for row in data]
```

The source is explicitly complete — **do not modify it**; the task is to
test it, configure packaging around it, and build it. `pytest` and
`build` are already installed; use `python3`, not `python`.

Required end state:
- `tests/test_predict.py` has ≥2 tests (one fraud case, one legitimate
  case) importing `predict` from `fraud_detection`; `pytest` passes.
- `pyproject.toml` has: `[build-system]` with
  `requires = ["setuptools>=61.0", "wheel"]` and
  `build-backend = "setuptools.build_meta"`; `name = "fraud_detection"`;
  `version = "0.1.0"`; `requires-python = ">=3.10"`;
  `dependencies = ["scikit-learn", "pandas", "numpy"]`;
  `[tool.pytest.ini_options]` with `pythonpath = ["src"]`.
- Building produces `dist/fraud_detection-0.1.0-*.whl`.

---

## 2. Reasoning model — how to *derive* the fix, not memorize it

### 2.1 Read the source and its docstring before writing anything

```bash
find src/fraud_detection -maxdepth 2 -type f -print
grep -R -n "def predict" src/fraud_detection
cat src/fraud_detection/*.py
```

The docstring states the rule explicitly: *"flags any transaction whose
first feature value exceeds 100."* This is the entire spec the tests need
to encode — nothing about `predict`'s internals matters, only its
observable input→output contract. Since the task says the source is
already complete, the job is strictly "write tests that pin down this
behavior," never "change the implementation until tests pass" — the same
"test the behavior, don't chase the test" discipline as Day 5's Makefile
lab, applied to Python instead of shell.

### 2.2 The exact boundary condition the tests must encode

```python
return [1 if row[0] > 100 else 0 for row in data]
```

The comparison is **strictly** `>`, not `>=`. This one character decides
what a test at exactly `100` must assert:

```text
amount = 150   → 150 > 100  → True  → 1  (fraud)
amount = 100   → 100 > 100  → False → 0  (legitimate — NOT fraud)
```

A test using `100` as the "legitimate" case isn't an arbitrary choice —
it's a deliberate **boundary test**, checking the exact edge where the
rule flips, rather than a comfortably-far value like `50`. A boundary
test catches the class of bug where someone later changes `>` to `>=` (or
vice versa) — a value like `50` would still pass either way, silently
missing that regression.

### 2.3 The `src/` layout is why "it's just a folder move" isn't true

```text
fraud-detection/
├── pyproject.toml
├── src/
│   └── fraud_detection/     ← the actual package
├── tests/
│   └── test_predict.py
└── dist/
```

Putting the package under `src/` instead of at the repo root is a common,
deliberate convention — it prevents accidentally importing the package
from the *source tree* instead of from its properly *installed* location,
a mistake that's easy to make with a flat layout and can hide packaging
bugs until they surface in production. The tradeoff: Python doesn't
automatically know `src/` is on the import path just because it exists —
that has to be told to whatever's doing the importing (here, `pytest`).

### 2.4 Reproducing and reading the actual failure, not guessing

```bash
pytest
```
```text
ModuleNotFoundError: No module named 'fraud_detection'
```

This error is **not** a bug in `predict()` — `pytest` is trying to
`import fraud_detection` and failing because `src/` isn't on
`sys.path` yet. Distinguishing "the code is wrong" from "the import path
is wrong" before touching anything is the same root-cause-before-fix
discipline as every prior lab — jumping straight to editing
`predict.py` here would be editing code that was explicitly declared
already correct.

### 2.5 `[tool.pytest.ini_options] pythonpath = ["src"]` — telling pytest where to look

```toml
[tool.pytest.ini_options]
pythonpath = ["src"]
```

This tells `pytest` specifically (not Python globally, not `pip`) to add
`src/` to its import search path before collecting and running tests —
so `from fraud_detection import predict` resolves during test runs even
though the package isn't installed and `src/` isn't on `PYTHONPATH` in
the ambient shell. This is a **test-runner-scoped** fix, distinct from
how the *built wheel* finds the package (§2.7) — two different mechanisms
solving two different "where is this package" problems for two different
consumers (pytest vs. an installed environment).

### 2.6 `[build-system]` — declaring what's needed to build this project at all

```toml
[build-system]
requires = ["setuptools>=61.0", "wheel"]
build-backend = "setuptools.build_meta"
```

```text
pyproject.toml
      │
      ▼
Build frontend (e.g. `python3 -m build`)
      │
      ▼
Build backend (setuptools.build_meta, per `build-backend` above)
      │
      ▼
Wheel / sdist
```

`requires` lists what must be installed *to run the build itself*
(distinct from the package's own runtime `dependencies`, §2.8);
`build-backend` names which tool actually knows how to turn source +
config into a wheel. Without a correct `[build-system]`, `python3 -m
build` has no way to know which backend to invoke at all.

### 2.7 `[project]` metadata — name, version, and why the wheel filename follows directly from it

```toml
[project]
name = "fraud_detection"
version = "0.1.0"
requires-python = ">=3.10"
dependencies = ["scikit-learn", "pandas", "numpy"]
```

The resulting wheel's filename is deterministically derived from exactly
these two fields:

```text
name = "fraud_detection"  ─┐
version = "0.1.0"         ─┴──►  fraud_detection-0.1.0-py3-none-any.whl
```

If the task requires a wheel named `fraud_detection-0.1.0-*.whl`, that
requirement is entirely satisfied (or violated) by these two config
lines — not by anything in the build command itself. `requires-python`
and `dependencies` don't affect the filename, but they're metadata the
wheel carries forward for anyone installing it later (§2.9).

### 2.8 Distribution name vs. import name — same trap as Day 3's `sklearn`/`scikit-learn`

```toml
dependencies = ["scikit-learn", "pandas", "numpy"]
```

`scikit-learn` here is a **distribution name** (what PyPI/pip installs),
even though the code that uses it would `import sklearn` — the same
distribution-name-vs-import-name split covered in the Day 3 lockfile lab.
`fraud_detection`, by contrast, happens to be both the distribution name
*and* the import name for this project (underscores in both) — but that's
a property of this specific package, not a rule that always holds.

### 2.9 What actually ends up inside the built wheel — verify, don't assume

```bash
unzip -l dist/fraud_detection-0.1.0-py3-none-any.whl
```
```text
fraud_detection/__init__.py
fraud_detection/predict.py
fraud_detection-0.1.0.dist-info/METADATA
fraud_detection-0.1.0.dist-info/WHEEL
fraud_detection-0.1.0.dist-info/top_level.txt
fraud_detection-0.1.0.dist-info/RECORD
```

"The build succeeded" and "the wheel exists" are not the same claim as
"the wheel contains the right files" — a misconfigured
`[tool.setuptools.packages.find]` could produce a wheel that builds
cleanly but is missing the actual package, or includes something
unintended. Opening the artifact and reading its contents is the actual
verification step, the same "don't trust the response, inspect the real
resource" discipline as every `describe-*` call in the AWS labs, applied
to a build artifact instead of a cloud resource.

### 2.10 Reading the wheel filename's own encoded meaning

```text
fraud_detection - 0.1.0 - py3 - none - any . whl
      │             │       │     │     │
      │             │       │     │     └── platform-independent (pure Python)
      │             │       │     └──────── no specific C ABI required
      │             │       └────────────── compatible with any Python 3.x
      │             └────────────────────── package version
      └──────────────────────────────────── distribution name
```

`py3-none-any` specifically confirms this is a **pure-Python wheel** — no
compiled extension modules, no OS/CPU-specific binary code — which is
exactly what you'd expect for a package containing only `.py` files
(`predict.py`, `__init__.py`).

### 2.11 Cleaning before rebuilding — why stale artifacts are a verification hazard

```bash
rm -rf dist build *.egg-info
python3 -m build
```

If an old wheel already sits in `dist/` from a previous (possibly wrong)
configuration, a fresh build might add to it rather than replace it, or
you might accidentally inspect/ship the stale one without realizing the
new config was never actually exercised. Cleaning first guarantees
whatever appears in `dist/` afterward was produced by *this* run, against
*this* configuration — a small habit that prevents a specific, easy false
"it worked" conclusion.

### 2.12 Not every build-time message is a failure — classify it

```text
warning: sdist: standard file not found: should have one of README, README.rst, README.txt, README.md
...
Successfully built fraud_detection-0.1.0.tar.gz and fraud_detection-0.1.0-py3-none-any.whl
```

The task's requirements never mention a README — this warning is
informational (setuptools has an opinion about best practice), not a
blocker against anything actually required here. The general discipline:
sort build output into **error** (must fix), **warning** (check whether
it affects an actual requirement), and **success confirmation** (still
verify the artifact directly, per §2.9) — don't treat "a warning
appeared" and "the build failed" as the same event.

### 2.13 The compressed reasoning chain

```text
Requirement (tests pass; pyproject.toml fields correct; wheel builds and contains the package)
   → Read predict.py's docstring/logic                → strict > 100, not >=
   → Write tests/test_predict.py                        → one fraud case (>100), one boundary case (==100)
   → Run pytest                                          → ModuleNotFoundError: fraud_detection
   → Diagnose: src/ layout isn't on pytest's import path → NOT a code bug
   → Add [tool.pytest.ini_options] pythonpath = ["src"]
   → Re-run pytest                                       → 2 passed
   → Fix [build-system], name, version, requires-python, dependencies in pyproject.toml
   → rm -rf dist build *.egg-info   (clean before rebuilding)
   → python3 -m build
   → unzip -l dist/*.whl                                 → verify actual contents, not just "build succeeded"
   → Confirm wheel filename matches fraud_detection-0.1.0-*.whl
```

---

## 3. Concepts (reference)

### 3.1 Unit test vs. implementation — testing behavior, not internals
A unit test should assert on a function's observable input→output
contract (what the docstring/spec promises), not on how it's
implemented internally. This is why the tests here call `predict(...)`
and check the returned list, rather than inspecting anything about how
the list comprehension works.

### 3.2 Boundary-value testing
Testing at the exact threshold where behavior changes (`100`, given a
strict `> 100` rule) catches a class of off-by-one/operator bugs
(`>` vs `>=`) that a comfortably-interior test value (like `50` or `500`)
would never catch, since both operators would still agree away from the
boundary.

### 3.3 `src/` layout vs. flat layout
`src/<package>/` (this project) forces tests and tooling to explicitly
resolve the package's location rather than accidentally picking it up
from the current working directory — a common, deliberate convention to
avoid packaging bugs that only manifest once the package is actually
installed elsewhere. The cost is exactly the `ModuleNotFoundError` this
lab walks through, and its fix (`pythonpath`).

### 3.4 `[build-system]` vs. `[project].dependencies`
`[build-system].requires` lists what's needed to **build** the package
(here, `setuptools`, `wheel`) — these are never installed alongside the
package for its *users*. `[project].dependencies` lists what's needed to
**run** the package (`scikit-learn`, `pandas`, `numpy`) — these get
recorded in the wheel's metadata and installed for anyone who later
`pip install`s it. Two different audiences, two different sections.

### 3.5 Wheel (`.whl`) vs. source distribution (`.tar.gz`)
A wheel is a pre-built, ready-to-install distribution — no build step
needed at install time. A source distribution (sdist) ships source plus
build metadata, requiring the build backend to run again at install
time. `python3 -m build` produces both by default; this task specifically
requires the wheel.

### 3.6 `.dist-info/` inside a wheel
- `METADATA` — package name, version, dependencies, Python requirement.
- `WHEEL` — wheel-format-specific metadata (which tool built it, tags).
- `RECORD` — every file in the wheel plus its hash/size, used to verify
  integrity and support clean uninstallation.

---

## 4. Runbook

### 4.1 Inspect the source before writing tests
```bash
cd /root/code/fraud-detection
find src/fraud_detection -maxdepth 2 -type f -print
```
```text
src/fraud_detection/__init__.py
src/fraud_detection/predict.py
```
```bash
grep -R -n "def predict" src/fraud_detection
cat src/fraud_detection/*.py
```
Confirms the rule: `row[0] > 100` → `1` (fraud), else `0`.

### 4.2 Write the unit tests
```bash
cat > tests/test_predict.py <<'EOF'
from fraud_detection import predict


def test_predicts_fraud_for_amount_above_100():
    result = predict([[150.0, 2.0, 3.0]])
    assert result == [1]


def test_predicts_legitimate_for_amount_at_or_below_100():
    result = predict([[100.0, 2.0, 3.0]])
    assert result == [0]
EOF
```

### 4.3 Run pytest — reproduce the import failure first
```bash
pytest
```
```text
ModuleNotFoundError: No module named 'fraud_detection'
```

### 4.4 Inspect and fix the packaging configuration
```bash
cat pyproject.toml
```
```toml
[project]
name = "fraud-detection"
version = "0.0.1"
requires-python = ">=3.8"
dependencies = []

[tool.setuptools.packages.find]
where = ["src"]
```
```bash
cat > pyproject.toml <<'EOF'
[build-system]
requires = ["setuptools>=61.0", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "fraud_detection"
version = "0.1.0"
description = "Fraud detection model for xFusionCorp Industries"
requires-python = ">=3.10"
dependencies = ["scikit-learn", "pandas", "numpy"]

[tool.setuptools.packages.find]
where = ["src"]

[tool.pytest.ini_options]
pythonpath = ["src"]
EOF
```

### 4.5 Re-run pytest — should now pass
```bash
pytest
```
```text
collected 2 items

tests/test_predict.py .. [100%]

2 passed in 0.05s
```

### 4.6 Clean any stale build artifacts
```bash
rm -rf dist build *.egg-info
```

### 4.7 Build the package
```bash
python3 -m build
```
```text
warning: sdist: standard file not found: should have one of README, README.rst, README.txt, README.md
...
Successfully built fraud_detection-0.1.0.tar.gz and fraud_detection-0.1.0-py3-none-any.whl
```

### 4.8 Verify the exact artifact exists
```bash
ls dist/
```
```text
fraud_detection-0.1.0-py3-none-any.whl
fraud_detection-0.1.0.tar.gz
```

### 4.9 Verify the wheel's actual contents, not just its existence
```bash
unzip -l dist/fraud_detection-0.1.0-py3-none-any.whl
```
```text
fraud_detection/__init__.py
fraud_detection/predict.py
fraud_detection-0.1.0.dist-info/METADATA
fraud_detection-0.1.0.dist-info/WHEEL
fraud_detection-0.1.0.dist-info/top_level.txt
fraud_detection-0.1.0.dist-info/RECORD
```

### 4.10 Submit
With `pytest` passing, `pyproject.toml` matching every required field,
and the wheel confirmed to contain the actual package, click **Check**
in the KodeKloud lab UI.

---

## 5. Final state

| Requirement | Result |
|---|---|
| `tests/test_predict.py` — fraud case | ✅ |
| `tests/test_predict.py` — legitimate/boundary case | ✅ |
| `pytest` passes | ✅ `2 passed` |
| `[build-system].requires` | ✅ `["setuptools>=61.0", "wheel"]` |
| `[build-system].build-backend` | ✅ `setuptools.build_meta` |
| `name` | ✅ `fraud_detection` |
| `version` | ✅ `0.1.0` |
| `requires-python` | ✅ `>=3.10` |
| `dependencies` | ✅ `["scikit-learn", "pandas", "numpy"]` |
| `[tool.pytest.ini_options].pythonpath` | ✅ `["src"]` |
| Wheel built | ✅ `dist/fraud_detection-0.1.0-py3-none-any.whl` |
| Wheel contains the package | ✅ verified via `unzip -l` |

```text
             fraud_detection
                    │
          ┌─────────┴─────────┐
          ▼                   ▼
       Source              Tests
    (unmodified)          (new, 2 tests)
          │                   │
          └─────────┬─────────┘
                    ▼
             pyproject.toml
        (build-system + project + pytest config)
                    │
                    ▼
             python3 -m build
                    │
                    ▼
 dist/fraud_detection-0.1.0-py3-none-any.whl
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `pytest` reports `collected 0 items` | Test file/function naming doesn't match pytest's discovery convention (`test_*.py`, `test_*` functions) | Confirm the file is `tests/test_predict.py` and functions start with `test_` |
| `ModuleNotFoundError: No module named 'fraud_detection'` | `src/` isn't on the import path pytest uses | Add `[tool.pytest.ini_options]` with `pythonpath = ["src"]` |
| Wheel filename doesn't match `fraud_detection-0.1.0-*.whl` | `name` or `version` in `[project]` doesn't match exactly (including underscore vs. hyphen) | Set `name = "fraud_detection"` and `version = "0.1.0"` precisely |
| `python3 -m build` fails immediately, before any packaging output | `[build-system]` missing or pointing at the wrong backend | Set `requires = ["setuptools>=61.0", "wheel"]` and `build-backend = "setuptools.build_meta"` |
| Old/stale wheel appears to satisfy the check despite a still-broken config | `dist/` had a leftover artifact from a previous, different configuration | `rm -rf dist build *.egg-info` before every rebuild you intend to verify |
| Built wheel is missing the actual package files | `[tool.setuptools.packages.find]` misconfigured relative to the `src/` layout | `unzip -l dist/*.whl` to check contents directly; confirm `where = ["src"]` |
| A build warning appears and it's unclear if it matters | Not every message at build time is a failure | Classify: does this warning correspond to an actual stated requirement? If not (e.g. missing README here), it's safe to ignore |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.13) out loud
      from "test and package this module" to "verified wheel with correct
      contents in `dist/`."
- [ ] Explain, in one sentence, why the legitimate-case test uses exactly
      `100` rather than a smaller value like `50`.
- [ ] Explain why `ModuleNotFoundError` here was a test-configuration
      problem, not a bug in `predict()`, given the task said the source
      was already complete.
- [ ] Explain the difference between `[build-system].requires` and
      `[project].dependencies` — who consumes each one, and when.
- [ ] Explain why "the build succeeded" isn't sufficient proof the wheel
      contains the right files, and what command actually proves it.
- [ ] Revert `pyproject.toml` to its broken starting state and delete the
      test file, then redo both from memory — verify with `pytest` exit
      status and `unzip -l` on the resulting wheel, not just printed text.

---

## 8. Document template for future lab days

Use this skeleton for `day-N/README.md`:

```markdown
# Day N — <task title>

## 1. Scenario
<what was asked, what was broken/required, constraints>

## 2. Reasoning model — how to derive the fix/commands
<walk the requirement down step by step, in the order you'd actually
discover it: reproduce the failure → diagnose each root cause
individually → fix → re-run → verify the actual success state. End with
a compressed arrow-chain version.>

## 3. Concepts (reference)
<one subsection per concept the task exercises — explain WHY, not just WHAT>

## 4. Runbook
<copy-pasteable commands in the order run, with real intermediate
output/results inline where they mattered to the diagnosis>

## 5. Final state
<table + diagram of what the configuration/artifact looks like after
completion>

## 6. Troubleshooting
<symptom / cause / fix table>

## 7. Do it yourself (checklist)
<"walk the reasoning chain out loud" + comprehension questions + "break it
again and fix it from memory" prompt>
```
