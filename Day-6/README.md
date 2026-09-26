# Day 6 — Fix a Broken Ruff and Black Configuration

A KodeKloud "100 MLOps" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive the fix
yourself*.

---

## 1. Scenario

The xFusionCorp ML team enforces code quality on every PR using `ruff`
(linting) and `black` (formatting). The project at
`/root/code/fraud-detection/` fails both checks.

Broken config found in `pyproject.toml`:

```toml
[tool.ruff]
line-length = 88
select = ["E", "F"]

[tool.black]
line-length = 100
```

Required end state:
- `ruff` and `black` both configured with a line length of **120**.
- Ruff's lint rule selection includes **`E`, `F`, `W`, `I`**.
- `ruff check src/` exits **0**.
- `black --check src/` exits **0**.

---

## 2. Reasoning model — how to *derive* the fix, not memorize it

### 2.1 Two different tools, checking two different things

```text
                 Python source
                      │
          ┌───────────┴───────────┐
          ▼                       ▼
        Ruff                    Black
       Linter                  Formatter
          │                       │
          ▼                       ▼
"Is something WRONG           "Is it formatted the
 with this code?"              way we've agreed on?"
```

**Ruff** is a linter — it looks for actual problems: unused imports,
undefined names, unsorted imports, common bugs. **Black** is an
opinionated formatter — it doesn't ask "is this correct," it asks "does
this match one canonical style" (line wrapping, quote style, spacing).
This distinction matters because a fix for one tool's failure often has
nothing to do with the other's — treating "both are red" as one problem
leads to wasted effort in the wrong place.

### 2.2 Reproduce both failures before touching anything

```bash
cd /root/code/fraud-detection
ruff check src/
black --check src/
```

Running both, separately, before changing anything tells you which tool
is unhappy about *what* — rather than guessing and fixing things that
might not even be broken. Same "observe → reproduce → understand" habit
as the JupyterLab lab (Day 2) and the broken Makefile lab (Day 5).

### 2.3 `pyproject.toml` as the single source of tool policy

Instead of every developer (and CI) remembering to type
`ruff check src/ --line-length 120` by hand, the project's policy is
declared once, in `pyproject.toml`:

```toml
[tool.ruff]
line-length = 120

[tool.ruff.lint]
select = ["E", "F", "W", "I"]

[tool.black]
line-length = 120
```

Both tools read their own `[tool.<name>]` table from the same file —
after which `ruff check src/` and `black --check src/`, with no extra
flags, apply exactly this policy. This is the same "config lives in one
declared file, tools read it" pattern as `requirements.in`/`uv.lock`
(Day 3) — the file is the single source of truth, not a flag repeated in
every invocation.

### 2.4 Diagnosing the broken config — two separate numeric mismatches

```text
                current           required
Ruff line-length    88        →      120
Black line-length   100        →      120
Ruff rule selection E, F       →      E, F, W, I
```

Three independent gaps, not one — worth listing them explicitly before
editing, the same way Day 3's broken `requirements.in` had three
independent root causes rather than "the file is just wrong."

### 2.5 Why Ruff's config needed restructuring, not just a number change

```toml
[tool.ruff]
line-length = 88
select = ["E", "F"]          ← this line is the problem
```

Running `ruff check` against this actually surfaces a deprecation
warning:

```text
warning: The `select` option in `[tool.ruff]` is deprecated. Use `[tool.ruff.lint]` instead.
'select' -> 'lint.select'
```

Modern Ruff versions moved lint-rule-specific settings (`select`,
`ignore`, per-rule options) into their own `[tool.ruff.lint]` table,
separate from Ruff's general settings (`line-length`, `target-version`,
etc.) which stay directly under `[tool.ruff]`. The warning is
*informative*, not noise — it's telling you exactly where the setting
belongs now:

```toml
[tool.ruff]
line-length = 120

[tool.ruff.lint]
select = ["E", "F", "W", "I"]
```

### 2.6 What the four rule codes actually check

```text
E  → pycodestyle errors     (general PEP 8-style violations)
F  → Pyflakes                (unused imports, undefined names, actual bugs)
W  → pycodestyle warnings    (softer style issues than E)
I  → isort-equivalent        (import statement ordering/grouping)
```

`E`, `F`, `W`, `I` are **rule family prefixes** — each Ruff diagnostic
code you'll see (`F401`, `E501`, `I001`, ...) starts with one of these
letters, telling you which family it belongs to before you even read the
message.

### 2.7 Enabling a new rule family can surface previously-invisible problems

Before `I` was enabled, Ruff only checked `E` and `F` — it had no opinion
on import ordering at all. The moment `I` was added to `select`, Ruff
started reporting:

```text
I001 Import block is un-sorted or un-formatted
```

This is not a new bug introduced by editing the config — the import
block was *always* unsorted; nothing was checking for it before. This is
a generalizable lesson: **tightening a linter's rule selection routinely
reveals pre-existing issues that were simply never being looked for** —
the config change didn't create the problem, it turned the light on.

### 2.8 The actual code-level bug Ruff's `F` rules caught

```python
import os
import pandas as pd
```

```text
F401 `os` imported but unused
```

`os` is genuinely never referenced anywhere else in the file — this is a
real defect (dead import), not a false positive or a config quirk. The
fix is to remove the line, not to suppress the rule:

```bash
sed -i '/^import os$/d' src/data/process_data.py
```

Suppressing `F401` instead (via a `# noqa` comment or an `ignore` list)
would hide the *symptom* while leaving genuinely dead code in place —
worth explicitly choosing "fix the code" over "silence the linter" when
the finding is real.

### 2.9 Ruff passing doesn't imply Black will pass — they're unrelated checks

```text
ruff check src/     → All checks passed!
black --check src/  → would reformat 1 file
```

Code can simultaneously be **lint-clean** (no unused imports, no
undefined names, sorted imports) and **not Black-formatted** (wrong
quote style, wrong line wrapping, wrong spacing) — the two tools check
entirely orthogonal properties of the same file. Passing one is never
evidence the other will pass; both must be checked and fixed
independently (§2.2).

### 2.10 `black src/` (mutates) vs. `black --check src/` (reports only)

```bash
black src/            # actually rewrites files to match Black's style
black --check src/    # asks "would anything change?" — exits nonzero if so, touches nothing
```

```text
black src/
        ↓
"1 file reformatted, 4 files left unchanged."   ← files were CHANGED

black --check src/
        ↓
"5 files would be left unchanged."              ← files were NOT touched,
                                                    this just confirms
                                                    they're already correct
```

This distinction is exactly why CI pipelines use `--check`, never the
bare form: CI's job is to *report* a violation and force a human to fix
it locally, not to silently rewrite someone's pushed code out from under
them. Local development workflow: run `black src/` to actually fix
formatting; CI workflow: run `black --check src/` to gate the PR.

### 2.11 The compressed reasoning chain

```text
Requirement (ruff + black both line-length 120; ruff rules E,F,W,I; both exit 0)
   → Reproduce: ruff check src/ && black --check src/     → both fail, see why
   → Diagnose: pyproject.toml has THREE gaps (§2.4)
   → Fix: line-length 88→120 (ruff), 100→120 (black)
   → Fix: move `select` into [tool.ruff.lint], add W and I
   → Re-run ruff check src/                                 → NEW finding: I001 (import order)
   → Re-run again                                            → F401 (unused `os` import)
   → Fix the SOURCE (remove unused import), not the rule selection
   → ruff check src/                                        → All checks passed!
   → black --check src/                                     → would reformat 1 file (still failing)
   → black src/  (locally, to actually apply the fix)
   → black --check src/                                     → 5 files would be left unchanged (passes)
```

---

## 3. Concepts (reference)

### 3.1 Linter vs. formatter
A linter (Ruff) flags things that are *semantically* questionable or
outright wrong — unused code, undefined names, likely bugs. A formatter
(Black) enforces a single, deterministic *style* — the code isn't wrong
without it, just inconsistent with the rest of the codebase. Both matter
for code quality, but they solve different problems and are configured
and invoked independently.

### 3.2 `pyproject.toml` as centralized tool configuration
A single TOML file where each tool reads its own namespaced table
(`[tool.ruff]`, `[tool.black]`, `[tool.pytest.ini_options]`, etc.) —
avoids scattering `.flake8`, `setup.cfg`, and tool-specific config files
across the repo, and avoids relying on developers remembering the right
CLI flags every time.

### 3.3 Ruff's `[tool.ruff]` vs `[tool.ruff.lint]` split
General Ruff behavior (line length, target Python version, file
exclusions) lives directly under `[tool.ruff]`. Anything specifically
about *which lint rules run* (`select`, `ignore`, rule-specific options)
lives under the nested `[tool.ruff.lint]` table. Ruff's own deprecation
warnings are a reliable signal when older-style config needs to move.

### 3.4 Ruff rule family prefixes
`E` (pycodestyle errors), `F` (Pyflakes — unused/undefined names, real
bugs), `W` (pycodestyle warnings), `I` (import sorting) are four of many
available rule families (others include `B` for bugbear, `UP` for
pyupgrade, etc.). `select = [...]` is an allow-list — only enabled
families' rules are checked at all.

### 3.5 Why a widened lint policy surfacing new failures is expected, not a regression
If code was never checked against a rule family, it can accumulate
violations invisibly. Turning that family on for the first time is
supposed to surface everything that's been wrong all along — the
correct response is fixing the surfaced issues, not narrowing the rule
selection back down to make the noise go away.

### 3.6 `--check` as the general CI-safe pattern
Many formatting/fixing tools offer both a mutating mode and a
check-only/dry-run mode (the same idea as `terraform plan` vs `apply`,
or `make -n` vs `make`, from Day 5). CI pipelines should default to the
non-mutating mode — its job is to gate on the *current* state, not
silently alter it.

---

## 4. Runbook

### 4.1 Reproduce both failures
```bash
cd /root/code/fraud-detection
ruff check src/
```
```text
src/data/process_data.py:1:8: F401 `os` imported but unused
Found 1 error.
```
```bash
black --check src/
```
```text
would reformat src/data/process_data.py
Oh no! 💥 💔 💥
1 file would be reformatted, 4 files would be left unchanged.
```

### 4.2 Inspect the current (broken) configuration
```bash
cat pyproject.toml
```
```toml
[tool.ruff]
line-length = 88
select = ["E", "F"]

[tool.black]
line-length = 100
```
Three gaps against the requirement: both line-lengths wrong, and
`select` missing `W`/`I` (§2.4).

### 4.3 Fix the Ruff and Black configuration
```bash
cat > pyproject.toml <<'EOF'
[project]
name = "fraud-detection"
version = "0.1.0"

[tool.ruff]
line-length = 120

[tool.ruff.lint]
select = ["E", "F", "W", "I"]

[tool.black]
line-length = 120
EOF
```

### 4.4 Re-run Ruff — a new finding appears
```bash
ruff check src/
```
```text
src/data/process_data.py:1:1: I001 [*] Import block is un-sorted or un-formatted
src/data/process_data.py:1:8: F401 [*] `os` imported but unused
Found 2 errors.
```
Enabling `I` surfaced the import-order issue that was always there but
never checked (§2.7).

### 4.5 Fix the actual source code, not the rule selection
```bash
sed -i '/^import os$/d' src/data/process_data.py
```

### 4.6 Verify Ruff passes
```bash
ruff check src/
```
```text
All checks passed!
```

### 4.7 Check Black — passing Ruff didn't fix Black (§2.9)
```bash
black --check src/
```
```text
would reformat src/data/process_data.py
Oh no! 💥 💔 💥
1 file would be reformatted, 4 files would be left unchanged.
```

### 4.8 Apply Black's formatting locally
```bash
black src/
```
```text
reformatted src/data/process_data.py
All done! ✨ 🍰 ✨
1 file reformatted, 4 files left unchanged.
```

### 4.9 Final verification — both tools, both exit 0
```bash
ruff check src/; echo "ruff exit: $?"
```
```text
All checks passed!
ruff exit: 0
```
```bash
black --check src/; echo "black exit: $?"
```
```text
All done! ✨ 🍰 ✨
5 files would be left unchanged.
black exit: 0
```

### 4.10 Submit
With both commands exiting `0` and `pyproject.toml` matching the
required policy (§4.3), click **Check** in the KodeKloud lab UI.

---

## 5. Final state

| Requirement | Result |
|---|---|
| Ruff line length `120` | ✅ |
| Black line length `120` | ✅ |
| Ruff rules include `E` | ✅ |
| Ruff rules include `F` | ✅ |
| Ruff rules include `W` | ✅ |
| Ruff rules include `I` | ✅ |
| `ruff check src/` exits `0` | ✅ |
| `black --check src/` exits `0` | ✅ |

```text
             Project Python Quality Policy
                         │
            ┌────────────┴────────────┐
            ▼                         ▼
           Ruff                      Black
      line-length = 120         line-length = 120
            │
      select = E, F, W, I
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `warning: The 'select' option in [tool.ruff] is deprecated` | Rule selection declared directly under `[tool.ruff]` instead of `[tool.ruff.lint]` | Move `select = [...]` into a `[tool.ruff.lint]` table |
| New errors appear right after widening `select` | The newly-enabled rule family (e.g. `I`) was never being checked before | Fix the surfaced issues in source — this is expected, not a config bug (§2.7) |
| `ruff check src/` passes but `black --check src/` still fails | Ruff and Black check unrelated properties of the code | Run and fix both independently; one passing is never evidence for the other |
| `black --check` reports failures that never go away after editing `pyproject.toml` | Config was fixed, but the actual formatting was never applied to the files | Run `black src/` (mutating) once locally, then re-verify with `black --check src/` |
| CI unexpectedly modifies a developer's pushed code | `black src/` (mutating form) used in CI instead of `black --check src/` | CI should always use `--check`; only local dev workflows should run the mutating form |
| Suppressed an `F401`/similar warning instead of fixing it | Reached for `# noqa` or an `ignore` list instead of removing genuinely dead code | Only suppress a rule for a real false positive; a real unused import should be deleted |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.11) out loud
      from "both `ruff check` and `black --check` fail" to "both exit 0."
- [ ] Explain, in one sentence, the difference between what Ruff checks
      and what Black checks.
- [ ] Explain why enabling the `I` rule family produced a *new* error
      instead of the config change itself being the bug.
- [ ] Explain why passing `ruff check src/` gave no guarantee that
      `black --check src/` would also pass.
- [ ] Explain the difference between `black src/` and
      `black --check src/`, and which one belongs in a CI pipeline.
- [ ] Revert `pyproject.toml` to the broken version and the unused
      `import os` line, then fix both from memory — verify with the exit
      code of each command, not just its printed message.

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
<table + diagram of what the configuration/infrastructure looks like
after completion>

## 6. Troubleshooting
<symptom / cause / fix table>

## 7. Do it yourself (checklist)
<"walk the reasoning chain out loud" + comprehension questions + "break it
again and fix it from memory" prompt>
```
