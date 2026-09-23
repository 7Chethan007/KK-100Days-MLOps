# Python Environments, `pip`, Virtual Environments & `uv` — A Hands-On KT Runbook

This is a **do-it-yourself** revision runbook, not just a read. Every
section has commands you can actually run (any machine with Python 3
works). The goal isn't to memorize `uv` commands — it's to understand
**what problem each layer of tooling solves**, so that `pip`, `venv`, and
`uv` all click into one coherent mental model instead of feeling like
three unrelated tools.

Companion doc: `README.md` in this folder applies `uv` specifically to
the fraud-detection lockfile lab. This file is the general-purpose KT.

---

## 0. Set up a sandbox (30 seconds)

```bash
mkdir -p ~/uv-kt && cd ~/uv-kt
python3 --version
which python3
```

Keep this terminal open — every example below is meant to be typed, not
just read.

---

## 1. The big picture — several different problems, one project

Building a Python project actually bundles several separate problems
together:

```text
                    Python Project
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
  Which Python?    Where do packages   Which exact
  interpreter?     live, isolated?     versions?
        │                │                │
     Python            venv            pip / uv
```

| Problem | Traditional tool | Modern `uv` approach |
|---|---|---|
| Python interpreter | System Python / `pyenv` | `uv python` |
| Isolated environment | `venv` | `uv venv` |
| Install packages | `pip` | `uv pip` |
| Resolve dependencies | pip's resolver | `uv`'s resolver |
| Lock dependencies | manual / `pip-tools` | `uv lock` / `uv pip compile` |
| Run project | `python` | `uv run` |
| Manage a whole project | several tools stitched together | `uv` |

The idea to hold onto:

> **`uv` isn't "a faster pip." It's a package *and* project manager that
> can cover the entire row above with one tool.**

---

## 2. What is Python itself?

```bash
python3 --version
```

```text
Python 3.12.3
```

This is the **interpreter** — the program that executes `.py` files and
ships a standard library. Try it:

```bash
echo 'print("hello from the interpreter")' > hello.py
python3 hello.py
```

Python ships plenty in its standard library (`os`, `json`, `math`,
`datetime`...) but nowhere near everything a real project needs — that
gap is what packages fill.

---

## 3. What is a package, and where does it come from?

A real project imports things the standard library doesn't have:

```python
import pandas
import numpy
```

**Try it — prove these aren't built in:**

```bash
python3 -c "import pandas"
```

```text
ModuleNotFoundError: No module named 'pandas'
```

`pandas` has to be fetched from somewhere and installed before Python can
find it. That "somewhere" is **PyPI** (the Python Package Index) —
Python's central public package repository — and the tool that fetches
from it is `pip`.

---

## 4. `pip` — Python's traditional installer

```bash
pip install pandas
```

```text
                 PyPI
                  │
                  │ download
                  ▼
                 pip
                  │
                  ▼
        Python environment
                  │
                  └── pandas (+ its own dependencies)
```

**Try it now:**

```bash
python3 -m venv .venv   # (we'll explain venv properly in §7 — needed first so this install is isolated)
source .venv/bin/activate
pip install pandas
python3 -c "import pandas; print(pandas.__version__)"
```

You should see a version number print — `pandas` is now importable,
because `pip` fetched it from PyPI and placed it somewhere Python's
import system looks.

---

## 5. What actually happens during `pip install pandas`

```text
pip install pandas
        │
        ▼
Find "pandas" on the package index
        │
        ▼
Determine a compatible version for your Python
        │
        ▼
Read pandas's OWN declared dependencies
        │
        ▼
Install pandas + everything it depends on
```

**Try it — see the dependency fan-out for yourself:**

```bash
pip show pandas
```

```text
Name: pandas
Version: 2.x.x
...
Requires: numpy, python-dateutil, pytz, tzdata
```

`Requires:` is pandas's own dependency list — installing one package
pulled in at least four others. This fan-out is exactly why a 4-package
`requirements.in` can resolve to 88 pinned packages (see the companion
lockfile lab).

---

## 6. Why does one global environment become a problem?

Imagine two projects on the same machine:

```text
Project A (fraud-detection, new)    Project B (legacy scoring model)
pandas 3.x                          pandas 2.x
scikit-learn 1.x                    scikit-learn 0.24
```

If both installed into the **same** global Python:

```text
                 System Python
                      │
          ┌───────────┴───────────┐
          ▼                       ▼
    Project A                 Project B
    needs pandas 3.x          needs pandas 2.x
          └───────────┬───────────┘
                       ▼
              only ONE pandas can
              be installed globally
                  at a time
```

One shared install location can't satisfy two projects that need
different, incompatible versions of the same package at the same time.
This is the exact problem virtual environments exist to solve.

---

## 7. Virtual environments — isolated package spaces

```text
                 Machine
                    │
             System Python
                    │
        ┌───────────┴────────────┐
        ▼                        ▼
     .venv-A                  .venv-B
        │                        │
    pandas 3.x               pandas 2.x
    scikit-learn 1.x          scikit-learn 0.24
```

Each project gets its own private package directory. Installing into one
never touches, upgrades, or breaks the other.

### Creating one traditionally

```bash
python3 -m venv .venv
```

```text
project/
├── .venv/
│   ├── bin/        (python, pip, activate, ...)
│   ├── lib/         (installed packages live here)
│   └── ...
└── app.py
```

Activate it:

```bash
source .venv/bin/activate
```

Your prompt changes to show you're inside it:

```text
(.venv) user@machine$
```

---

## 8. What activation *actually* does — try it, don't assume

This is the single most-misunderstood step in the whole workflow.
Activation does **not** reinstall Python or magically switch
interpreters — it just edits your shell's `PATH` for the current session.

**Try it — inspect `PATH` before and after:**

```bash
deactivate 2>/dev/null   # make sure we start deactivated
which python3
```

```text
/usr/bin/python3
```

```bash
source .venv/bin/activate
which python3
```

```text
/Users/you/uv-kt/.venv/bin/python3
```

```text
Before activation                    After activation
PATH:                                PATH:
  /usr/local/bin                       .venv/bin        ← inserted FIRST
  /usr/bin                             /usr/local/bin
  ...                                  /usr/bin
                                        ...
```

Because `.venv/bin` is now searched *first*, typing `python` or `pip`
resolves to the venv's copies instead of the system ones. This is why
`which python` / `which pip` is the fastest way to answer *"which
environment am I actually in right now?"* whenever something behaves
unexpectedly.

```bash
deactivate
which python3
```

```text
/usr/bin/python3     ← back to system Python; nothing was ever "installed" or "uninstalled"
```

---

## 9. The full traditional workflow, end to end

```bash
mkdir -p ~/uv-kt/classic-project && cd ~/uv-kt/classic-project
python3 -m venv .venv
source .venv/bin/activate

pip install pandas numpy scikit-learn
```

Freeze what's now installed into a file another machine can replay:

```bash
pip freeze > requirements.txt
cat requirements.txt
```

```text
numpy==2.x.x
pandas==2.x.x
scikit-learn==1.x.x
scipy==1.x.x
joblib==1.x.x
threadpoolctl==3.x.x
...
```

On another machine, someone reproduces this exactly:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

This works, but notice something important for the next section: nothing
here distinguished *"pandas is what I directly asked for"* from
*"scipy is just along for the ride because scikit-learn needs it."*
`pip freeze` flattens that distinction away.

---

## 10. `pip freeze` vs. a real `requirements.in` — two different questions

| | Question it answers | Example |
|---|---|---|
| `pip freeze` | "What is installed in this environment **right now**?" | `numpy==2.5.3`, `pandas==3.0.6`, ... |
| `requirements.in` | "What does my project **directly, intentionally** depend on?" | `numpy`, `pandas` (bare, no versions) |

```text
requirements.in
     ↓
PROJECT INTENT (small, hand-maintained, human-authored)

pip freeze output
     ↓
ENVIRONMENT STATE (large, machine-generated, includes everything)
```

**Try it — see the gap directly:**

```bash
wc -l requirements.txt        # from pip freeze above — likely 6+ lines
echo -e "pandas\nnumpy\nscikit-learn" > requirements.in
wc -l requirements.in          # exactly 3 lines — your actual intent
```

`pip freeze` is a snapshot of a *particular environment at a particular
moment* — it doesn't tell you which of those packages you actually
imported versus which just came along transitively. This gap is exactly
what tools like `pip-tools` and `uv pip compile` were built to close (see
the companion lockfile-lab README for the full worked example).

---

## 11. Where `pip` + `venv` stop being enough

A real Python project's full lifecycle touches all of these:

```text
Python version management
virtual environment creation
dependency resolution
lockfile generation
package installation
running code inside the right environment
```

Traditionally that's several separate tools stitched together:

```text
pyenv  (Python versions)
  +
venv   (isolation)
  +
pip    (installation)
  +
pip-tools  (compiling/locking requirements)
```

Each one works, but you're manually keeping them in sync. This is the gap
`uv` was built to close.

---

## 12. What is `uv`?

`uv` is a single, fast (Rust-implemented) tool that covers Python version
management, virtual environments, package installation, dependency
resolution, locking, and running code — one CLI instead of four.

```text
                  uv
                   │
       ┌───────────┼────────────┐
       │           │            │
       ▼           ▼            ▼
    Python       venv        packages
    versions     creation    install
       │           │            │
       └───────────┼────────────┘
                   │
              dependencies
              + resolution
              + locking
              + running
```

The mental model to keep: not *"uv replaces pip,"* but *"uv can replace
the whole pyenv + venv + pip + pip-tools stack, and does most of it
faster."*

**Try it:**

```bash
uv --version
```

If it's not installed: `curl -LsSf https://astral.sh/uv/install.sh | sh`
(or see the official install docs — don't blindly pipe scripts from
memory into `sh` without checking the source first).

---

## 13. `uv pip` — the deliberately pip-compatible interface

If you already know `pip`, this part requires almost no new learning.

```bash
pip install pandas          # traditional
uv pip install pandas       # uv, same interface, same PyPI target
```

**Try the side-by-side comparison yourself:**

```bash
cd ~/uv-kt && mkdir uv-project && cd uv-project
uv venv
source .venv/bin/activate
uv pip install pandas
python3 -c "import pandas; print(pandas.__version__)"
```

`uv venv` creates the exact same kind of `.venv/` directory `python -m
venv` does — you can still `source .venv/bin/activate` and use it exactly
like before. `uv` deliberately kept this interface familiar so migrating
from `pip`/`venv` doesn't require relearning everything at once.

---

## 14. Full comparison table: traditional vs. `uv`

| Task | Traditional | `uv` |
|---|---|---|
| Create venv | `python -m venv .venv` | `uv venv` |
| Activate | `source .venv/bin/activate` | same command still works |
| Install a package | `pip install pandas` | `uv pip install pandas` |
| Install exact version | `pip install pandas==3.0.6` | `uv pip install pandas==3.0.6` |
| Install from file | `pip install -r requirements.txt` | `uv pip install -r requirements.txt` |
| Compile a lockfile | `pip-compile` (separate tool) | `uv pip compile requirements.in -o requirements.txt` |
| Match env exactly to a file | no direct built-in equivalent | `uv pip sync requirements.txt` |
| List installed packages | `pip list` | `uv pip list` |
| Snapshot environment | `pip freeze` | `uv pip freeze` |
| Run a script in-env | activate, then `python app.py` | `uv run python app.py` (no activation needed) |
| Manage Python versions | `pyenv install 3.12` | `uv python install 3.12` |
| New project scaffold | manual | `uv init myproject` |
| Add a dependency to project | edit file by hand | `uv add pandas` |
| Lock a whole project | no single built-in tool | `uv lock` |
| Reproduce a project env | `pip install -r requirements.txt` | `uv sync` |

---

## 15. `uv run` — skipping manual activation

```bash
source .venv/bin/activate
python3 app.py
deactivate
```

vs.

```bash
uv run python3 app.py
```

**Try it:**

```bash
echo 'import pandas; print("pandas OK:", pandas.__version__)' > app.py
deactivate 2>/dev/null
uv run python3 app.py
```

`uv run` finds the project's environment, makes sure dependencies are
present, and runs your command inside it — all in one step, without you
needing to remember to activate first (or forgetting to, and silently
running against system Python instead).

---

## 16. Two different `uv` workflows — don't mix them up

This is the part most people conflate, so it's worth its own section.

### Workflow A — requirements files (what the fraud-detection lab uses)

```text
requirements.in
       ↓
   uv pip compile
       ↓
requirements.txt
```

You hand-maintain `requirements.in`; `uv` resolves it into a pinned
`requirements.txt`. This is the closer-to-`pip` workflow — good for
scripts, simple deployments, or teams already using requirements files.

### Workflow B — full project management

```text
pyproject.toml
       ↓
    uv lock
       ↓
    uv.lock
```

**Try it:**

```bash
cd ~/uv-kt && uv init fraud-detection-demo
cd fraud-detection-demo
cat pyproject.toml
```

```toml
[project]
name = "fraud-detection-demo"
version = "0.1.0"
dependencies = []
```

```bash
uv add pandas numpy scikit-learn
cat pyproject.toml
```

```toml
dependencies = [
    "pandas>=...",
    "numpy>=...",
    "scikit-learn>=...",
]
```

```bash
ls
```

```text
pyproject.toml   uv.lock   .venv/   main.py
```

`uv add` did three things at once: updated `pyproject.toml`, resolved and
wrote `uv.lock`, and installed into `.venv/` — the requirements-file
workflow needs three separate manual steps to do the same thing.

**Rule of thumb:** requirements files (`requirements.in`/`.txt`) for
simple scripts or when a lab/tool specifically expects them (like the
fraud-detection lockfile task); `pyproject.toml`/`uv.lock` for anything
you'd call "a project" with its own metadata, entry points, and build
config.

---

## 17. `pyproject.toml` — what it actually holds

```toml
[project]
name = "fraud-detection"
version = "0.1.0"
dependencies = [
    "numpy",
    "pandas",
    "scikit-learn",
    "mlflow",
]
```

It's not just a dependency list — it's the project's metadata file
(name, version, entry points, build system config, tool settings for
`black`/`ruff`/`pytest`, etc.). Dependencies are one section of it, not
the whole point of the file. `uv lock` reads the `[project]` section's
`dependencies` and resolves them into `uv.lock`, the same way
`uv pip compile` reads `requirements.in`.

---

## 18. `uv pip install` vs `uv pip sync` — a distinction that matters in CI

```bash
uv pip install -r requirements.txt
```

means *"make sure these are installed"* — it adds/upgrades what's listed,
but leaves anything else already in the environment untouched.

```bash
uv pip sync requirements.txt
```

means *"make the environment match this file exactly"* — it also
**removes** anything installed that isn't in the file.

**Try it — see `sync` actually remove something:**

```bash
uv pip install requests          # install something NOT in requirements.txt
uv pip list | grep requests      # confirm it's there
uv pip sync requirements.txt     # sync back to only what's declared
uv pip list | grep requests      # gone
```

```text
Current environment                    requirements.txt (declared)
────────────────────                   ───────────────────────────
pandas, numpy, requests    --sync-->   pandas, numpy
                                        (requests removed — not declared)
```

This is why CI pipelines should generally use `sync`, not `install` —
you want the exact declared set every run, not an environment that
silently accumulates whatever got installed manually at some point.

---

## 19. Why is `uv` fast? (just enough to explain it to someone else)

- Written in Rust, not Python — less interpreter overhead for the tool
  itself.
- Aggressive caching — a package downloaded once is reused across
  projects/environments instead of re-fetched every time.
- Parallelized downloads and installs, instead of one-at-a-time.
- A resolver architecture built for speed from the start, rather than
  pip's resolver which was retrofitted onto an older design.

For MLOps purposes, the practical payoff is: **the same workflow you
already know from `pip`, running noticeably faster in CI, which matters
when a pipeline resolves and installs dependencies on every run.**

---

## 20. What each layer actually solves — the separation that matters

```text
              Python interpreter
                    │
           "run Python code at all"
                    │
                    ▼
                  venv
                    │
             "isolate packages
              per-project"
                    │
                    ▼
              pip / uv pip
                    │
           "install packages
            from PyPI"
                    │
                    ▼
           dependency resolver
                    │
          "find one version set
           that satisfies every
           package's constraints"
                    │
                    ▼
                lockfile
                    │
          "record the exact
           resolved versions,
           reproducibly"
```

A common misconception worth naming directly: **a venv does not, by
itself, guarantee reproducibility.** Isolation and pinning are two
different problems — a venv keeps Project A's packages from colliding
with Project B's, but says nothing about whether two different machines,
each creating their own fresh venv from a loose `pandas` (no version)
spec, end up with the *same* version. That's the lockfile's job, not the
venv's.

---

## 21. Practical MLOps shape (project workflow, end to end)

```text
fraud-detection/
│
├── pyproject.toml     ← direct dependency intent + project metadata
├── uv.lock            ← fully resolved, pinned dependency graph
├── .venv/             ← isolated environment (gitignored)
├── src/
│   ├── train.py
│   └── predict.py
└── tests/
```

```bash
uv lock              # resolve pyproject.toml → uv.lock
uv sync              # (in CI) reproduce the exact locked environment
uv run pytest        # run tests inside that environment
uv run python src/train.py
```

```text
Developer machine → uv.lock committed to git
        │
        ▼
CI pipeline → uv sync (reproduces the SAME resolved graph)
        │
        ▼
Training job → identical dependency versions as CI
        │
        ▼
Production inference → identical dependency versions as training
```

That end-to-end identical-versions chain is the actual MLOps payoff — not
"uv is fast," but "uv makes it hard to accidentally drift."

---

## 22. Command reference — memorize these, understand the rest

```bash
# Python itself
python3 --version
which python3

# Traditional venv
python3 -m venv .venv
source .venv/bin/activate
deactivate

# pip
pip install <package>
pip install <package>==<version>
pip install -r requirements.txt
pip list
pip show <package>
pip freeze

# uv — pip-compatible interface
uv venv
uv pip install <package>
uv pip install -r requirements.txt
uv pip compile requirements.in -o requirements.txt
uv pip sync requirements.txt
uv pip list
uv pip freeze

# uv — project workflow
uv init <project>
uv add <package>
uv remove <package>
uv lock
uv sync
uv run <command>
uv python install <version>
uv --version
```

---

## 23. Concept table — what to actually understand, not memorize

| Concept | Mental model |
|---|---|
| Python interpreter | The runtime that executes `.py` files |
| PyPI | The public repository packages are downloaded from |
| `venv` | Per-project isolation — stops projects' package versions colliding |
| `pip` | Installs packages from PyPI into the active environment |
| Dependency | A package your project or a library it uses needs |
| Transitive dependency | A dependency of your dependency, not something you asked for directly |
| Resolver | Finds one version set that satisfies every package's constraints simultaneously |
| `requirements.in` | Hand-authored, direct dependency intent |
| `requirements.txt` | Machine-generated, fully resolved and pinned |
| `pip freeze` | A snapshot of an environment's current state — not the same as intent |
| `pyproject.toml` | Project metadata, including its declared dependencies |
| `uv.lock` | Fully resolved project dependency graph, keyed off `pyproject.toml` |
| `uv` | One tool spanning Python versions, venvs, installs, resolution, locking, running |

---

## 24. Self-test — do these without looking above

1. Explain, in one sentence, what `source .venv/bin/activate` actually
   changes in your shell. Verify your answer with `which python3` before
   and after.
2. Why doesn't creating a virtual environment, by itself, guarantee two
   machines get identical package versions?
3. What's the difference between what `pip freeze` outputs and what a
   well-maintained `requirements.in` contains?
4. You run `uv pip install requests` then `uv pip sync requirements.txt`
   where `requests` isn't in that file. What happens to `requests`, and
   why would you want that behavior in a CI pipeline?
5. What's the difference between the requirements-file workflow
   (`requirements.in` → `uv pip compile` → `requirements.txt`) and the
   project workflow (`pyproject.toml` → `uv lock` → `uv.lock`)? When would
   you reach for each?
6. Without running it, predict what `uv run python3 app.py` does
   differently from `python3 app.py` run without activating any
   environment first.

---

## 25. The one-sentence model to carry forward

> **`venv` isolates a project's packages from every other project on the
> machine; `pip` (or `uv pip`) installs packages from PyPI into that
> isolated space; a resolver figures out one mutually compatible version
> set across every direct and transitive dependency; a lockfile
> (`requirements.txt` or `uv.lock`) records that exact resolved set so any
> other machine can reproduce it byte-for-byte; and `uv` is a single fast
> tool that does all of the above, plus Python version management and
> project scaffolding, instead of stitching together `pyenv` + `venv` +
> `pip` + `pip-tools` separately.**
