# Day 5 — Fix a Broken ML Workflow Makefile

A KodeKloud "100 MLOps" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive it yourself*.

---

## 1. Scenario

The xFusionCorp ML team uses a `Makefile` at
`/root/code/fraud-detection/Makefile` to standardize data processing,
training, testing, and cleanup — but `make all` doesn't complete
successfully.

Broken Makefile found:

```makefile
# fraud-detection Makefile

setup:
    python3 -m venv mlops-venv && mlops-venv/bin/pip install -r requirements.txt

data:
    python3 src/data/process_data.py

train:
    python3 src/models/train.py

test:
    pytest tests/

clean:
    rm -rf __pycache__

all: setup train test
```

(Note: every recipe line above is shown with leading spaces for display —
in the actual broken file, the `data`/`train`/`test`/`clean` lines were
genuinely space-indented, not tab-indented, which is the root cause in
§2.2.)

Required end state:
- Six targets exist with the specified behavior: `setup`, `data`,
  `train`, `test`, `clean`, `all`.
- `clean` recursively removes every `__pycache__` directory, removes
  `.pytest_cache`, and clears the contents of `models/`.
- `all` runs `setup`, `data`, `train`, `test` — **in that order**.
- All six targets are declared `.PHONY`.
- Every recipe line is indented with a real **tab** character, not
  spaces.
- `make all` completes without error.

---

## 2. Reasoning model — how to *derive* the fix, not memorize it

### 2.1 Reproduce the failure before touching anything

```bash
cd /root/code/fraud-detection
make all
```

Deliberately running the broken Makefile first — rather than jumping
straight to rewriting it — tells you *which* line Make is actually
complaining about, instead of guessing at problems that might not even be
the one causing the failure. This is the same "observe → reproduce →
understand → fix" discipline used in the JupyterLab lab (Day 2).

### 2.2 Reading the failure: `missing separator`

```text
Makefile:7: *** missing separator.  Stop.
```

This is Make's characteristic error for exactly one specific mistake:
**a recipe line indented with spaces instead of a tab**. Make's file
format is rigid on this point — a line under a target is only recognized
as part of that target's recipe if it starts with a literal tab
character; anything else (including 4 or 8 spaces that *look* identical
in an editor) is treated as a syntax error, not "close enough."

```text
setup:
<TAB>python3 -m venv mlops-venv ...    ← valid recipe line

data:
    python3 src/data/process_data.py   ← invalid: starts with spaces,
                                           not a tab, even though it
                                           LOOKS identically indented
```

This is a genuinely easy trap: most editors render a tab and 4-8 spaces
identically on screen, so the bug is invisible by eye — you have to
either know to check, or let Make's error message tell you.

### 2.3 Confirming the invisible: `cat -A` reveals tabs vs. spaces

```bash
cat -A Makefile
```

`cat -A` (equivalent to `cat -vet`) shows non-printing characters
explicitly — a tab renders as `^I`, and a trailing space/line-end as `$`.
This turns an invisible formatting bug into something you can literally
point at:

```text
setup:$
^Ipython3 -m venv mlops-venv ...$        ← ^I = tab, correct
data:$
    python3 src/data/process_data.py$    ← spaces visible, no ^I, broken
```

Any editor's "show whitespace" mode does the same job — the point is
never *trust* that indentation "looks like a tab," always *verify* it
when debugging a Makefile.

### 2.4 Problem 2 — `all` runs the wrong target order

```makefile
all: setup train test
```

Required order: `setup`, `data`, `train`, `test`. The broken file skips
`data` entirely — meaning `train` would run against whatever (possibly
stale, possibly absent) data happened to already exist, not data freshly
produced by this run. In an ML pipeline specifically, silently training on
stale data is a much more dangerous bug than a missing shell command: it
doesn't crash, it just quietly produces a wrong or misleading model.

### 2.5 Problem 3 — `clean` doesn't clean what the task requires

```makefile
clean:
    rm -rf __pycache__
```

Two gaps against the requirement:
1. `rm -rf __pycache__` only removes a `__pycache__` directory sitting in
   the **current** directory — it does not touch
   `src/fraud_detection/__pycache__/` or any other nested one, because
   `rm -rf` has no concept of "recursively search for directories named
   X" on its own.
2. `.pytest_cache` and the contents of `models/` aren't touched at all.

The fix needs a **search**, not just a **delete**:

```bash
find . -type d -name '__pycache__' -prune -exec rm -rf {} +
```

```text
find .                        → search starting from the current directory
  -type d                     → only match directories
  -name '__pycache__'         → ...named exactly __pycache__
  -prune                      → don't descend INTO a matched directory
                                  (no point searching inside something
                                   you're about to delete wholesale)
  -exec rm -rf {} +           → run rm -rf on all matches, batched
                                  ({} + passes many paths to one rm call,
                                   rather than one rm call per match)
```

This is the same `find`-driven pattern as recursive permission fixes
(Day 4's DevOps lab) — "locate everything matching a pattern, anywhere in
the tree" is `find`'s job; a bare `rm -rf` only ever sees the one path you
give it.

### 2.6 Problem 4 — missing `.PHONY`

```makefile
.PHONY: setup data train test clean all
```

Make's default behavior assumes a target name refers to a **file** it
should produce — if a file literally named `test` or `clean` ever existed
in the project directory, and that file were newer than its
prerequisites, Make would decide the target is "already up to date" and
silently skip running the recipe at all. `.PHONY` tells Make: *"these
names are actions, never files — always run the recipe, regardless of
what's on disk."* Every target in this Makefile represents an action
(run a script, install packages, delete files), never a file it's meant
to *produce*, so all six belong in `.PHONY`.

### 2.7 `all: setup data train test` is a dependency list, not a shell one-liner

```makefile
all: setup data train test
```

This does **not** mean "run one command with four arguments." It means
"before doing anything else for `all`, make sure targets `setup`, `data`,
`train`, and `test` have each been run, in that order, left to right."
`all` itself has no recipe body — its entire job is expressing this
ordering:

```text
                 all
                  │
       ┌────┬─────┼─────┐
       ▼    ▼     ▼     ▼
     setup data train  test
```

If any one of the four fails, Make stops the chain by default rather than
continuing on to the next — a training run's test suite never gets a
chance to hide a real setup/data failure behind an unrelated pass.

### 2.8 Dry-run before actually running anything destructive

```bash
make -n all
```

`-n` ("no-op"/dry-run) prints every command Make *would* execute, without
executing any of them:

```text
make all       →  actually run everything
make -n all    →  show the execution plan only
```

The same "plan before apply" instinct as `terraform plan`, `kubectl
--dry-run`, or `git commit --dry-run` — worth reaching for whenever you've
just edited automation and want to sanity-check the *sequence* before
committing to actually running it (especially here, since `setup`
provisions a venv and `train` may take real time).

### 2.9 The compressed reasoning chain

```text
Requirement (6 correct targets, .PHONY, correct ordering, make all succeeds)
   → Reproduce: make all                          → "missing separator" at line 7
   → Diagnose: cat -A Makefile                      → spaces where a tab is required
   → Diagnose: all: setup train test                → missing "data" in the chain
   → Diagnose: clean only rm -rf's one __pycache__   → needs find, + .pytest_cache, + models/*
   → Diagnose: no .PHONY line at all                 → add it, listing all six targets
   → Rewrite Makefile with real tabs, correct clean, correct all, .PHONY
   → make -n all                                     → confirm the planned command sequence
   → make all                                        → runs setup → data → train → test, succeeds
```

---

## 3. Concepts (reference)

### 3.1 Target / prerequisite / recipe — the three parts of every Make rule
```makefile
target: prerequisite1 prerequisite2
	recipe line 1
	recipe line 2
```
- **Target** — the name you invoke (`make train`) or another rule
  depends on.
- **Prerequisites** — other targets that must run first (`all`'s entire
  body, in this lab).
- **Recipe** — the actual shell commands executed for that target; each
  line **must** start with a tab.

### 3.2 Why Make specifically requires a tab, not spaces
This is a long-standing, famously sharp edge in Make's file format: the
tab character is how Make's parser distinguishes "this line continues the
previous target's recipe" from "this line starts something new." There is
no configuration to relax this in standard Make — the fix is always to
make the indentation an actual tab, not to adjust Make's tolerance.

### 3.3 `.PHONY`
A special built-in target whose prerequisites are declared as "not real
files." Any target listed under `.PHONY` always runs its recipe when
invoked, regardless of whether a same-named file exists or how recently
it was modified. Almost every target in a task-runner-style Makefile
(as opposed to a Makefile that's genuinely compiling files) should be
`.PHONY`.

### 3.4 `find -exec ... {} +` vs. a bare `rm -rf`
`rm -rf path` deletes exactly the path(s) you name, nothing more. `find
... -exec rm -rf {} +` first **searches** the tree for everything
matching a pattern (by name, type, depth, etc.), then deletes each match.
Whenever a cleanup task says "every X anywhere in the project," that's
`find`'s job, not a single hardcoded `rm -rf`.

### 3.5 `make -n` (dry-run)
Prints the resolved command sequence for a target without executing it —
useful for validating a Makefile edit's *ordering and expansion* before
committing to a real (possibly slow, possibly destructive) run.

### 3.6 Why `mlops-venv/bin/pip`, not just `pip`
```makefile
setup:
	python3 -m venv mlops-venv && mlops-venv/bin/pip install -r requirements.txt
```
Explicitly calling `mlops-venv/bin/pip` guarantees packages install into
*this* freshly-created venv, regardless of what `pip`/`python` currently
resolve to in the invoking shell's `PATH` — the same "don't trust an
ambient activated environment" caution as the `uv`/`venv` guide in Day 3.

---

## 4. Runbook

### 4.1 Reproduce the failure
```bash
cd /root/code/fraud-detection
make all
```
```text
Makefile:7: *** missing separator.  Stop.
```

### 4.2 Inspect the Makefile, including invisible whitespace
```bash
cat -n Makefile
cat -A Makefile
```
Confirms: `data`, `train`, `test`, and `clean`'s recipe lines are
space-indented (no `^I`), and `all`/`clean`/`.PHONY` don't match the
required behavior (§2.2–§2.6).

### 4.3 Inspect the project layout (know what the recipes actually target)
```bash
find . -maxdepth 3 -type d | sort
```
```text
.
./configs
./data
./data/processed
./data/raw
./models
./notebooks
./src
./src/data
./src/features
./src/models
./src/utils
./tests
```
Confirms `src/data/process_data.py`, `src/models/train.py`, `tests/`, and
`models/` are real, expected paths for the recipes to reference.

### 4.4 Rewrite the Makefile
```bash
cat > Makefile <<'EOF'
.PHONY: setup data train test clean all

setup:
	python3 -m venv mlops-venv && mlops-venv/bin/pip install -r requirements.txt

data:
	python3 src/data/process_data.py

train:
	python3 src/models/train.py

test:
	pytest tests/

clean:
	find . -type d -name '__pycache__' -prune -exec rm -rf {} +
	rm -rf .pytest_cache
	rm -rf models/*

all: setup data train test
EOF
```

**Critical:** every recipe line above (`python3 -m venv ...`, `python3
src/data/...`, `python3 src/models/...`, `pytest tests/`, and all three
lines of `clean`) must begin with a literal **tab**, typed as an actual
tab keystroke — not 4 or 8 spaces that merely display the same way in a
heredoc or editor (§2.2, §2.3).

### 4.5 Confirm the fix — tabs are actually present
```bash
cat -A Makefile | grep -E '^\^I' | head
```
Every recipe line should show a leading `^I` (tab), not leading spaces.

### 4.6 Dry-run before executing
```bash
make -n all
```
```text
python3 -m venv mlops-venv && mlops-venv/bin/pip install -r requirements.txt
python3 src/data/process_data.py
python3 src/models/train.py
pytest tests/
```
Confirms the exact right four commands, in the exact right order —
`setup → data → train → test` — before actually running anything.

### 4.7 Run the full workflow
```bash
make all
```
Runs `setup`, then `data`, then `train`, then `test`, stopping immediately
if any one of them fails.

### 4.8 Verify `clean` independently
```bash
make clean
find . -type d -name '__pycache__'    # should print nothing
ls .pytest_cache 2>&1                 # should report "No such file or directory"
ls models/                            # should be empty
```

### 4.9 Verify `.PHONY` and target list
```bash
grep '^\.PHONY' Makefile
```
```text
.PHONY: setup data train test clean all
```

### 4.10 Submit
With `make all` completing cleanly and every requirement above verified,
click **Check** in the KodeKloud lab UI.

---

## 5. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `Makefile:N: *** missing separator.  Stop.` | A recipe line is indented with spaces instead of a tab | `cat -A Makefile`, find the line missing `^I`, retype it with a real tab |
| Editing the Makefile in an editor keeps "helpfully" converting tabs to spaces | Editor's auto-indent/format-on-save setting | Disable tab-to-space conversion for this file, or use a heredoc (`cat > Makefile <<'EOF' ... EOF`) from the shell instead |
| `make all` runs but `train` uses stale/wrong data | `all`'s prerequisite list skips `data` | `all: setup data train test` — include every stage, in order |
| `make clean` doesn't remove nested `__pycache__` directories | Bare `rm -rf __pycache__` only matches the current directory | Use `find . -type d -name '__pycache__' -prune -exec rm -rf {} +` |
| A stray file named `test` or `clean` causes `make` to skip the recipe entirely | Target not declared `.PHONY`, so Make treats it as a file-based target that's "already up to date" | Add all task-runner-style targets to `.PHONY` |
| `make -n all` shows commands in the wrong order or missing one | `all`'s prerequisite list itself is wrong, independent of any tab issue | Fix the `all:` line directly; `-n` only reflects what's actually declared |

---

## 6. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.9) out loud
      from "`make all` fails" to "verified via `make -n all` and a clean
      `make all` run."
- [ ] Explain, in one sentence, why `Makefile:7: missing separator` almost
      always means "check for spaces where a tab belongs."
- [ ] Explain why `all: setup train test` (missing `data`) is a
      dangerous bug specifically in an ML pipeline, not just a cosmetic
      one.
- [ ] Explain what `.PHONY` protects against, using a concrete example of
      a file that could otherwise break `make test`.
- [ ] Explain the difference between `rm -rf __pycache__` and
      `find . -type d -name '__pycache__' -prune -exec rm -rf {} +`.
- [ ] Break the Makefile again (revert one recipe line back to
      space-indentation) and fix it from memory using only `cat -A` to
      diagnose, not this file.

---

## 7. Document template for future lab days

Use this skeleton for `day-N/README.md`:

```markdown
# Day N — <task title>

## 1. Scenario
<what was asked, what was broken/required, constraints>

## 2. Reasoning model — how to derive the fix/commands
<walk the requirement down step by step, in the order you'd actually
discover it: reproduce the failure → diagnose each root cause
individually → fix → re-run in dry-run mode → verify the actual success
state. End with a compressed arrow-chain version.>

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
