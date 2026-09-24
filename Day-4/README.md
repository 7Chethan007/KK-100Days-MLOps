# Day 4 — Add a `.gitignore` and Untrack Committed Artifacts

A KodeKloud "100 MLOps" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive it yourself*.

---

## 1. Scenario

The xFusionCorp `fraud-detection` repository was committed before any
`.gitignore` existed, so generated and local artifacts got swept into
version control alongside real source code:

```text
.env                                              ← local secrets
venv/pyvenv.cfg                                   ← virtual environment
src/fraud_detection/__pycache__/model.cpython-312.pyc  ← Python bytecode cache
models/fraud_model.pkl                            ← trained model file
.ipynb_checkpoints/analysis-checkpoint.ipynb       ← notebook checkpoint
```

Required end state:
- A `.gitignore` at the repo root excludes `__pycache__/`, `*.pyc`,
  `venv/`, `.ipynb_checkpoints/`, `*.pkl`, `.env`.
- Those already-tracked artifacts are removed from **Git's index** (not
  disk), and that cleanup is committed.
- The actual project sources stay tracked: everything under
  `src/fraud_detection/`, `README.md`, `requirements.txt`.

The task explicitly warns: *"a `.gitignore` never untracks files Git
already tracks"* — that single sentence is the entire lab.

---

## 2. Reasoning model — how to *derive* the fix, not memorize it

### 2.1 Three separate places a file can "exist"

```text
Working tree  →  the actual files sitting on disk
Index         →  Git's staging area: what will go into the NEXT commit
Repository    →  committed history, stored in .git/
```

A file that's already been committed is simultaneously "yes" in all
three:

```text
Disk        ✅
Index       ✅
Repository  ✅  (as of the commit that added it)
```

`.gitignore` only ever affects one question: *"if this path is
currently untracked, should `git add`/`git status` notice it?"* It has
no mechanism to reach backward and remove something already sitting in
the index or in history. This is the entire reason the task's cleanup
step exists — writing the file alone provably cannot satisfy the
requirement.

### 2.2 Inspect before touching anything

```bash
cd /root/code/fraud-detection
git status
git ls-files
```

`git ls-files` is the ground truth for "what does Git currently track,"
independent of what's on disk or what `.gitignore` says. In this lab it
showed the five artifacts already committed — confirming the diagnosis in
§2.1 before writing a single line of `.gitignore`.

### 2.3 What each ignored category actually is, and why it doesn't belong in Git

```text
__pycache__/, *.pyc   → compiled bytecode, regenerated automatically by
                          the interpreter on any machine — committing it
                          is committing a build artifact, not source
venv/                  → an entire local Python installation + packages;
                          environment-specific, and reproducible from
                          requirements.txt instead (see the companion
                          uv/pip/venv guide in Day-3)
.ipynb_checkpoints/    → Jupyter's autosave snapshots, generated state
*.pkl                  → a serialized trained model — a build OUTPUT of
                          the training code, not the code itself
.env                   → local secrets/config (API keys, DB URLs); if
                          committed, they're in Git history forever, even
                          if deleted in a later commit
```

The pattern underneath all five: **Git should hold the recipe (source,
config, dependency specs), not what the recipe produces or what's
specific to one machine.** That's the same "generated vs. source"
distinction from prior labs (compiled `.pyc` vs `.py`, `.venv` vs
`requirements.in`) applied at the repository-policy level instead of the
filesystem level.

### 2.4 Writing `.gitignore` only handles half the problem

```gitignore
__pycache__/
*.pyc
venv/
.ipynb_checkpoints/
*.pkl
.env
```

Committing this file changes behavior for **future** untracked files
only. Proving that distinction matters more than writing the file:

```bash
git check-ignore --no-index -v \
  .env models/fraud_model.pkl venv/pyvenv.cfg \
  .ipynb_checkpoints/analysis-checkpoint.ipynb \
  src/fraud_detection/__pycache__/model.cpython-312.pyc
```

`--no-index` asks *"would these rules match this path, ignoring whether
Git currently tracks it?"* — deliberately bypassing the "already
tracked" special case so you can validate pattern-matching in isolation
from the untracking problem. All five matched their intended rule, which
confirms the `.gitignore` syntax is correct — and simultaneously proves
nothing has actually been untracked yet, since `git ls-files` still shows
them (§2.2).

### 2.5 `git rm --cached` — the actual untracking operation

```text
git rm --cached <path>
        =
"Stop tracking this path in Git.
 Do NOT touch the file on disk."
```

```text
Before                          After `git rm --cached` + commit
Disk        ✅                  Disk        ✅   (unchanged)
Index       ✅                  Index       ❌
Repository  ✅ (old commit)     New commit  ❌   (removed going forward)
```

The old commit that originally added the file still has it in history —
`git rm --cached` doesn't rewrite history, it just stops carrying the
file forward from this commit onward. (Purging it from *all* history
entirely is a different, much more invasive operation — not what this
task asks for.)

### 2.6 Two ways to reach the same untracked state

**Targeted** — name exactly what to remove:

```bash
git rm -r --cached .env .ipynb_checkpoints/ models/fraud_model.pkl venv/ \
  src/fraud_detection/__pycache__/
```

**Bulk** — clear the whole index, then let `.gitignore` decide what
comes back:

```bash
git rm -r --cached .
git add .
```

```text
git rm -r --cached .
        ↓
   index emptied (disk untouched)
        ↓
      git add .
        ↓
.gitignore is now consulted for every path
        ↓
only non-ignored paths get re-staged
```

The bulk approach is what actually happened in this lab's transcript —
worth understanding *why* it's safe: `git rm --cached` never touches the
working tree, so "emptying the index" is fully recoverable by re-adding,
and `.gitignore` (already committed in the prior step, §2.4) is what
filters the re-add. Order matters here: `.gitignore` must exist and be
committed **before** running `git add .`, or the ignored files would just
get re-staged.

### 2.7 Why the bulk `git rm -r --cached .` output looked alarming

```text
rm '.env'
rm 'README.md'
rm 'requirements.txt'
rm 'src/fraud_detection/model.py'
...
```

Every tracked file appears to be "removed" — because `--cached` genuinely
does clear all of them from the index, including the ones you want to
keep. That's expected and reversible: the very next `git add .` restores
everything **except** whatever `.gitignore` now excludes. Reading `rm` in
this output as "deleted from my machine" instead of "removed from the
index" is the exact panic-inducing misread this section exists to head
off.

### 2.8 Verify twice — the index, and the disk, separately

```bash
git ls-files          # should show ONLY the intended tracked files
ls -la .env models/fraud_model.pkl venv .ipynb_checkpoints   # should still exist on disk
```

Two independent checks because they answer two independent questions —
"did Git stop tracking it" and "is my local copy intact" are not the same
fact, and confirming only one leaves the other unverified.

### 2.9 The full task rhythm

```text
1. Inspect    — git status / git ls-files (what's tracked right now?)
2. Diagnose   — realize .gitignore alone can't fix already-tracked files
3. Write      — .gitignore, commit it as its own step
4. Validate   — git check-ignore --no-index -v (rules match intended paths)
5. Untrack    — git rm -r --cached . && git add . (or targeted git rm --cached)
6. Verify     — git ls-files (index correct) + ls -la (disk unchanged)
7. Commit     — the cleanup, as a second, separate commit
```

Two commits, not one, mirrors two distinct concerns: "here is our ignore
policy going forward" vs. "here is us retroactively applying it to what
was already committed" — worth keeping separate in history even though
Git wouldn't force you to.

---

## 3. Concepts (reference)

### 3.1 Working tree vs. index vs. repository
Three distinct states Git tracks for every path — see §2.1. Most
"confusing" Git behavior (including this entire lab) resolves once you
identify which of the three states an operation actually changes.
`git rm --cached` touches only the index; a plain `rm` touches only the
working tree; a commit moves the index into the repository's history.

### 3.2 `.gitignore` pattern syntax used here
```text
__pycache__/    → a directory named exactly this, anywhere in the tree
*.pyc           → any file ending in .pyc, anywhere
venv/           → a directory named venv, anywhere
.ipynb_checkpoints/  → same directory-match pattern
*.pkl           → any file ending in .pkl, anywhere
.env            → an exact filename match
```
A trailing `/` restricts a pattern to directories; a leading `*` matches
any filename with that suffix, at any depth, unless anchored with a
leading `/`.

### 3.3 `git check-ignore -v` vs `git check-ignore --no-index -v`
Without `--no-index`, `check-ignore` factors in whether Git already
tracks the path (a tracked file can behave differently). `--no-index`
answers the pattern-matching question in isolation — useful specifically
when you need to validate `.gitignore` syntax against paths that are
*currently* tracked, exactly the situation in §2.4.

### 3.4 `git rm --cached` vs plain `git rm`
`git rm <path>` removes the file from both the index **and** the working
tree (deletes it from disk). `git rm --cached <path>` removes it from the
index only, leaving the working-tree copy untouched. For "untrack but
keep the file" tasks, `--cached` is not optional — the plain form would
destroy the trained model file, the `.env`, and the venv this task
explicitly says to keep.

### 3.5 Why secrets in Git history are a standing risk, not a solved problem
Deleting `.env` in a later commit does not remove it from earlier commits
still reachable in history — anyone with clone access can check out an
old commit and read the original secret. The correct posture is
"never commit it in the first place" (this lab's `.gitignore`); if a real
secret was ever committed, the actual fix is rotating the credential, not
just removing the file going forward.

### 3.6 Why ML artifacts (models, datasets) generally don't belong in Git
Git is optimized for diffing and merging text; large binary blobs like
`.pkl` model files bloat repository size permanently (even after
`git rm --cached`, old blobs remain in history) and diff meaninglessly.
The common real-world pattern is: Git holds training *code* and
*config*; a model registry, object store, or tool like MLflow/DVC holds
the model *artifacts* the code produces — see §4 of the
`uv-pip-venv-guide.md` companion doc in Day-3 for the same "recipe vs.
output" split applied to dependencies.

---

## 4. Runbook

### 4.1 Inspect current tracked state
```bash
cd /root/code/fraud-detection
git status
```
```text
On branch main
Untracked files:
  (use "git add <file>..." to include in what will be committed)
        .gitignore

nothing added to commit but untracked files present (use "git add" to track)
```

```bash
git ls-files
```
```text
.env
.ipynb_checkpoints/analysis-checkpoint.ipynb
README.md
models/fraud_model.pkl
requirements.txt
src/fraud_detection/__init__.py
src/fraud_detection/__pycache__/model.cpython-312.pyc
src/fraud_detection/model.py
venv/pyvenv.cfg
```
Confirms the diagnosis: five artifacts that should never have been
tracked are sitting right alongside real source files.

### 4.2 Write `.gitignore`
```bash
cat > .gitignore <<'EOF'
__pycache__/
*.pyc
venv/
.ipynb_checkpoints/
*.pkl
.env
EOF
```

### 4.3 Commit the ignore policy as its own step
```bash
git add .gitignore
git commit -m "Add gitignore"
```
```text
[main 034bce8] Add gitignore
 1 file changed, 6 insertions(+)
 create mode 100644 .gitignore
```

### 4.4 Validate the patterns against the actual tracked paths
```bash
git check-ignore --no-index -v \
  .env \
  models/fraud_model.pkl \
  venv/pyvenv.cfg \
  .ipynb_checkpoints/analysis-checkpoint.ipynb \
  src/fraud_detection/__pycache__/model.cpython-312.pyc
```
```text
.gitignore:6:.env       .env
.gitignore:5:*.pkl      models/fraud_model.pkl
.gitignore:3:venv/      venv/pyvenv.cfg
.gitignore:4:.ipynb_checkpoints/        .ipynb_checkpoints/analysis-checkpoint.ipynb
.gitignore:1:__pycache__/       src/fraud_detection/__pycache__/model.cpython-312.pyc
```
Every path matched the intended rule and line number — the `.gitignore`
syntax is correct. `git ls-files` at this point still shows all five,
confirming they remain tracked (§2.4) — this step alone was not the fix.

### 4.5 Empty the index, then rebuild it through `.gitignore`
```bash
git rm -r --cached .
```
```text
rm '.env'
rm '.gitignore'
rm '.ipynb_checkpoints/analysis-checkpoint.ipynb'
rm 'README.md'
rm 'models/fraud_model.pkl'
rm 'requirements.txt'
rm 'src/fraud_detection/__init__.py'
rm 'src/fraud_detection/__pycache__/model.cpython-312.pyc'
rm 'src/fraud_detection/model.py'
rm 'venv/pyvenv.cfg'
```
Every tracked path is listed — expected, see §2.7. Nothing on disk was
touched.

```bash
git add .
git status
```
```text
On branch main
Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
        deleted:    .env
        deleted:    .ipynb_checkpoints/analysis-checkpoint.ipynb
        deleted:    models/fraud_model.pkl
        deleted:    src/fraud_detection/__pycache__/model.cpython-312.pyc
        deleted:    venv/pyvenv.cfg
```
Exactly the five artifacts show as staged deletions — the "recipe" files
(`README.md`, `requirements.txt`, `.gitignore`, `src/fraud_detection/*`)
were silently re-added by `git add .`, since they aren't matched by any
`.gitignore` rule.

### 4.6 Commit the cleanup
```bash
git commit -m "Stop tracking generated and local artifacts"
```
```text
[main ...] Stop tracking generated and local artifacts
 5 files changed, 5 deletions(-)
 delete mode 100644 .env
 delete mode 100644 .ipynb_checkpoints/analysis-checkpoint.ipynb
 delete mode 100644 models/fraud_model.pkl
 delete mode 100644 src/fraud_detection/__pycache__/model.cpython-312.pyc
 delete mode 100644 venv/pyvenv.cfg
```

### 4.7 Verify the index — only intended files remain tracked
```bash
git status
```
```text
On branch main
nothing to commit, working tree clean
```

```bash
git ls-files
```
```text
.gitignore
README.md
requirements.txt
src/fraud_detection/__init__.py
src/fraud_detection/model.py
```
Exactly the required set: `.gitignore`, `README.md`, `requirements.txt`,
and everything under `src/fraud_detection/`.

### 4.8 Verify the disk — the files still physically exist
```bash
ls -la .env models/fraud_model.pkl venv .ipynb_checkpoints
```
```text
-rw-r--r-- 1 root root   35 Sep 24 10:39 .env
-rw-r--r-- 1 root root   17 Sep 24 10:39 models/fraud_model.pkl

.ipynb_checkpoints:
-rw-r--r-- 1 root root   14 Sep 24 10:39 analysis-checkpoint.ipynb

venv:
-rw-r--r-- 1 root root   16 Sep 24 10:39 pyvenv.cfg
```
Confirms the split the task actually asked for:
```text
Git tracking      ❌  (all five artifacts)
Local filesystem  ✅  (all five artifacts, untouched)
```

### 4.9 Click "Check" in the lab UI to validate.

---

## 5. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Artifacts still appear in `git ls-files` after writing `.gitignore` | `.gitignore` was added, but the already-tracked files were never explicitly untracked | Run `git rm -r --cached .` (or targeted `git rm --cached <path>`), then `git add .` and commit |
| `git rm --cached` also deleted the file from disk | Ran plain `git rm` instead of `git rm --cached` | Restore from the previous commit (`git checkout HEAD~1 -- <path>`) or recreate the file; re-run with `--cached` |
| `git rm --cached` (bulk form) errors with "No pathspec was given" | Ran `git rm --cached` with no path/argument (e.g. from a broken multi-line paste) | Supply an explicit path, or use `git rm -r --cached .` for the whole tree |
| `git add .` re-stages a file you meant to ignore | The pattern doesn't actually match that path, or `.gitignore` wasn't committed before running `git add .` | Re-check with `git check-ignore --no-index -v <path>`; confirm `.gitignore` itself is committed first |
| `git commit` says "nothing to commit, working tree clean" unexpectedly | The untracking step (`git rm --cached` + `git add .`) was never actually run, or was already committed in a prior step | `git ls-files` to check current tracked state before assuming the step is done |
| Real secret was committed before `.gitignore` existed | `.env` had already been pushed/shared before this cleanup | Removing it from the index is not sufficient — rotate the actual credential (§3.5); consider it compromised |

---

## 6. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.9) out loud
      from "artifacts were committed with no `.gitignore`" to "index and
      disk both verified correct."
- [ ] Explain, in one sentence, why writing a `.gitignore` alone cannot
      remove a file that's already tracked.
- [ ] Explain the difference between `git rm <path>` and
      `git rm --cached <path>`, and which one this task required.
- [ ] Explain why `git rm -r --cached .` followed by `git add .` is safe,
      even though its output lists every tracked file as `rm`.
- [ ] Explain why `.gitignore` and the untracking cleanup were committed
      as two separate commits rather than one.
- [ ] Recreate the scenario from scratch (commit a throwaway `.env` and
      `__pycache__/` to a test repo with no `.gitignore`, then fix it) —
      verify with both `git ls-files` and `ls -la`, not just one.

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
