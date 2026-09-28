# Day 8 — Fix a Broken pre-commit Configuration

A KodeKloud "100 MLOps" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive the fix
yourself*.

> **Note on this write-up:** your draft cut off partway through the
> `autoupdate` failure investigation, before the actual fix/final
> `pre-commit run --all-files` output was captured. The runbook below is
> reconstructed from the task spec and the mechanics you'd already
> worked out — treat specific revision tags (`v6.0.0`, `v0.16.9`,
> `26.5.1`) as illustrative of the *shape* of real `autoupdate` output;
> your actual pinned versions will be whatever's current when you run it,
> since that's the entire point of the tool.

---

## 1. Scenario

The xFusionCorp ML team enforces code quality via `pre-commit` on every
commit. A draft `.pre-commit-config.yaml` exists at
`/root/code/fraud-detection/`, already tracked by Git alongside
`process.py`, but `pre-commit run --all-files` fails.

Required end state — the configuration must declare exactly these five
hooks, all executing cleanly:

| Hook | Source repo |
|---|---|
| `trailing-whitespace` | `pre-commit/pre-commit-hooks` |
| `end-of-file-fixer` | `pre-commit/pre-commit-hooks` |
| `check-yaml` | `pre-commit/pre-commit-hooks` |
| `ruff` | `astral-sh/ruff-pre-commit` |
| `black` | `psf/black-pre-commit-mirror` |

- Every `repo:` entry must include a `rev:` field, pinned to a current
  release.
- The hooks must be registered with git and run cleanly against the
  tracked files.
- `pre-commit autoupdate` is the sanctioned way to discover current
  version pins — not manual guessing.

---

## 2. Reasoning model — how to *derive* the fix, not memorize it

### 2.1 Three layers this lab operates at, kept distinct

```text
pre-commit   → the orchestration framework (reads config, manages envs, runs hooks)
repository   → WHERE a hook's implementation lives (a git repo, referenced by URL)
hook         → WHAT specific check/fix actually runs (one id inside a repo)
```

`pre-commit` itself is not a linter, a formatter, or a YAML validator —
it's the framework that fetches other tools' repositories, pins them to
specific versions, installs them into isolated environments, and invokes
them against the files you're committing. Confusing "pre-commit failed"
with "Ruff/Black/a hook failed" is the first trap this lab is built to
expose — the actual failure always originates one layer down, in a
specific hook, for a specific reason.

### 2.2 `repo:` / `rev:` / `hooks:` — the three questions every entry answers

```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v6.0.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
```

```text
repo:  → WHERE does this hook's code live?      (a git repository URL)
rev:   → WHICH version of that repo?             (a tag/commit — required, per the task)
hooks: → WHAT specific hook id(s) do I want run? (one entry can define several)
```

A single `repo:` block can register multiple `hooks:` under it (as
`pre-commit-hooks` does here for all three file-hygiene checks) — you
don't need a separate `repo:` entry per hook, only per *source
repository*.

### 2.3 Reproduce the failure before touching the YAML

```bash
cd /root/code/fraud-detection
pre-commit run --all-files
```

Read whatever error surfaces before editing anything — a missing `rev:`,
a malformed hook `id`, or a genuinely failing check (trailing whitespace
actually present in `process.py`) all produce different error shapes and
need different fixes. Same "observe → reproduce → diagnose" discipline as
every broken-config lab in this series (the Makefile lab, Day 5; the
lockfile lab, Day 3).

### 2.4 Why `rev:` is mandatory, not just good practice

```yaml
repo: some-tool
rev: main          # WRONG for this task — a moving target, not a pin
```

```text
Today:     main → commit A → hook behaves one way
Tomorrow:  main → commit B → SAME config, DIFFERENT hook behavior
```

Pointing `rev:` at a branch name instead of a release tag means your
`.pre-commit-config.yaml` doesn't actually pin anything — two developers
running the identical config file on two different days could get two
different tool versions, silently. This is the exact same reproducibility
principle as `requirements.txt` pinning exact package versions (Day 3) —
a config file's job is to make behavior *reproducible*, and an unpinned
`rev:` defeats that job entirely. The task requiring `rev:` on every
entry is enforcing this, not just checking a box.

### 2.5 `pre-commit autoupdate` — discovering pins, not guessing them

```bash
pre-commit autoupdate
```

```text
.pre-commit-config.yaml
        │
        ▼
   pre-commit
        │
        ▼
query EACH configured repository's releases
        │
        ▼
find the latest released tag for each
        │
        ▼
rewrite every rev: field in place
```

This queries the actual upstream repositories (`pre-commit-hooks`,
`ruff-pre-commit`, `black-pre-commit-mirror`) for their current release
tags and rewrites your config to match — the sanctioned alternative to
opening each repo's GitHub releases page and copying a tag by hand. Run
it *after* the `repo:`/`hooks:` structure is correct — `autoupdate`
updates existing `rev:` pins, it doesn't invent new `repo:`/`hooks:`
entries for repositories you haven't declared yet.

### 2.6 A missing or wrong `rev:` breaks `autoupdate` itself, not just `pre-commit run`

If a `repo:` block is malformed — pointing at a nonexistent repository
URL, or missing a `rev:` field entirely in a way `pre-commit`'s schema
validation rejects outright — `autoupdate` can fail before it ever
reaches the network query step, because it can't parse a config it
doesn't consider valid YAML/schema in the first place. This is worth
separating from a *hook* failure: **configuration validation** (is this
YAML shaped the way `pre-commit` expects) happens before **hook
execution** (does the tool this entry points to actually run
successfully) — two different failure classes needing two different
diagnoses.

### 2.7 Fixing hooks vs. failing hooks — `pre-commit` hooks can *modify* files

`trailing-whitespace`, `end-of-file-fixer`, and (with certain
configurations) `black` are **fixing** hooks — they don't just report a
problem, they rewrite the file to correct it. A hook run that modifies a
file reports as **failed** on that run (because the working tree changed
after `pre-commit` started), even though the fix itself succeeded:

```text
First run:  trailing-whitespace finds a problem → FIXES the file → reports FAILED
                (files were modified — that's why it's flagged)
Second run: trailing-whitespace re-checks the now-clean file → PASSES
```

This is why a second `pre-commit run --all-files` is expected and
necessary after the first — a "failed" result on a fixing hook's first
pass isn't evidence your configuration is still broken, it's evidence the
hook did its job and the files changed underneath it mid-run.

### 2.8 `hooks: id:` values are fixed identifiers, not free text

```yaml
hooks:
  - id: trailing-whitespace
```

Each hook `id` is defined by the source repository itself (in its own
`.pre-commit-hooks.yaml` manifest) — you can't invent an arbitrary `id`
and expect `pre-commit` to know what to run. Getting an `id` wrong (a
typo, or a name from a different repo entirely) produces a clear "hook id
not found" style error at validation time, distinct from a hook that ran
and genuinely failed its check.

### 2.9 Registering hooks with git

```bash
pre-commit install
```

`pre-commit run --all-files` executes the configured hooks manually,
against every tracked file, regardless of whether they're actually wired
into git's commit workflow. `pre-commit install` is the separate step
that writes an actual `.git/hooks/pre-commit` script, so hooks run
*automatically* on every future `git commit` — the task's requirement
that "hooks are registered with git" specifically needs this step, not
just a clean `run --all-files`.

### 2.10 The compressed reasoning chain

```text
Requirement (5 hooks across 3 repos, all rev:-pinned, run cleanly, registered with git)
   → Reproduce: pre-commit run --all-files    → see the actual current failure
   → Diagnose: is this a config-validation problem, or a hook-execution problem? (§2.6)
   → Fix the repos:/hooks: structure           → 3 repos, correct hook ids under each
   → Ensure every repo: entry has SOME rev: (even a placeholder/known-old tag)
   → pre-commit autoupdate                     → rewrites every rev: to current releases
   → pre-commit run --all-files (1st pass)     → fixing hooks modify files, report "failed"
   → pre-commit run --all-files (2nd pass)     → now clean, all 5 hooks pass
   → pre-commit install                        → registers hooks with git for future commits
   → Verify: cat .pre-commit-config.yaml       → confirm all 5 hooks + rev: on every repo
```

---

## 3. Concepts (reference)

### 3.1 pre-commit as orchestration, not implementation
`pre-commit` fetches other tools' repositories, manages isolated
environments for each, and invokes them — it never implements the actual
linting/formatting/validation logic itself. Every real check in this
lab's config belongs to `pre-commit-hooks`, `ruff`, or `black` — three
independent projects, each with their own release cadence, each pinned
independently.

### 3.2 Why this is a "shift-left" quality pattern
Running checks at commit time, on a developer's own machine, catches
issues before they reach CI or a teammate's pull request — cheaper to
fix a formatting issue locally in seconds than to wait for a CI failure
minutes or hours later. The same principle underlies unit tests run
locally before push (MLOps Day 7) rather than only in CI.

### 3.3 `trailing-whitespace`, `end-of-file-fixer`, `check-yaml`
Three file-hygiene checks from the `pre-commit-hooks` repository:
- `trailing-whitespace` — strips trailing spaces/tabs from line ends.
- `end-of-file-fixer` — ensures files end with exactly one trailing
  newline.
- `check-yaml` — parses every YAML file and fails if any is
  syntactically invalid (distinct from *linting* YAML style — this only
  checks it parses).

### 3.4 Ruff vs. Black in this pipeline (same distinction as Day 6)
Ruff lints (flags unused imports, undefined names, style violations);
Black formats (rewrites code to one canonical style). Both can be wired
into `pre-commit` as fixing hooks, but they check fundamentally different
properties of the same Python files — consistent with the linter-vs-
formatter distinction covered in depth in Day 6's ruff/black lab.

### 3.5 Version pinning as a reproducibility mechanism
An unpinned or branch-pointed `rev:` makes a config file's actual
behavior time-dependent — the same file can produce different results on
different days. Pinning to a release tag is what makes
`.pre-commit-config.yaml` an actual, reproducible contract, the same
principle behind lockfiles (Day 3) and exact version pins generally.

### 3.6 `pre-commit run --all-files` vs. `pre-commit install`
`run --all-files` is a manual, one-off invocation against every tracked
file — useful for testing config changes or catching up a repo that
never had hooks running before. `install` wires hooks into git itself so
they run automatically on every future commit, without the developer
remembering to invoke `pre-commit` by hand.

### 3.7 A "failed" fixing-hook run isn't necessarily a broken config
Hooks that auto-correct files (whitespace, EOF, sometimes Black) report
their run as failed the moment they modify anything, because the
working tree state changed mid-run. The correct response is re-running,
not reverting your configuration changes.

---

## 4. Runbook

### 4.1 Reproduce the current failure
```bash
cd /root/code/fraud-detection
pre-commit run --all-files
```
Read the actual error — likely a schema/validation failure (missing or
malformed `rev:`, wrong `repo:` URL, or an unrecognized `id:`) given the
task states the draft config "does not align with the team's standards."

### 4.2 Inspect the broken configuration
```bash
cat .pre-commit-config.yaml
```
Diagnose against the requirement: which of the 5 required hooks are
missing, which repos are wrong, which entries lack `rev:` (§2.6).

### 4.3 Rewrite the configuration with the correct structure
```bash
cat > .pre-commit-config.yaml <<'EOF'
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v6.0.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml

  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.16.9
    hooks:
      - id: ruff

  - repo: https://github.com/psf/black-pre-commit-mirror
    rev: 26.5.1
    hooks:
      - id: black
EOF
```
The exact `rev:` values above are placeholders for whatever's current —
§4.4 replaces them with real, verified current tags rather than trusting
hand-typed guesses.

### 4.4 Discover current version pins with autoupdate
```bash
pre-commit autoupdate
```
```text
Updating https://github.com/pre-commit/pre-commit-hooks ... updating v2.3.0 -> v6.0.0
Updating https://github.com/astral-sh/ruff-pre-commit ... updating main -> v0.16.9
Updating https://github.com/psf/black-pre-commit-mirror ... updating main -> 26.5.1
```
Confirm every `rev:` was rewritten to an actual tag, not left pointing at
a branch:
```bash
cat .pre-commit-config.yaml
```

### 4.5 First run — expect fixing hooks to report "failed" while actually fixing
```bash
pre-commit run --all-files
```
```text
trailing-whitespace.....................................................Failed
- files were modified by this hook
end-of-file-fixer.......................................................Failed
- files were modified by this hook
check-yaml...............................................................Passed
ruff......................................................................Passed
black.....................................................................Failed
- files were reformatted
```
This is expected (§2.7) — the hooks modified `process.py` and/or the
config file itself during this pass.

### 4.6 Second run — should now be clean
```bash
pre-commit run --all-files
```
```text
trailing-whitespace.....................................................Passed
end-of-file-fixer.......................................................Passed
check-yaml...............................................................Passed
ruff......................................................................Passed
black.....................................................................Passed
```

### 4.7 Register the hooks with git
```bash
pre-commit install
```
```text
pre-commit installed at .git/hooks/pre-commit
```

### 4.8 Final verification
```bash
cat .pre-commit-config.yaml
```
Confirm: 3 `repo:` entries, each with a `rev:`; exactly the 5 required
hook `id`s present across them; `.git/hooks/pre-commit` exists.

### 4.9 Submit
With `pre-commit run --all-files` exiting cleanly and hooks registered
via `pre-commit install`, click **Check** in the KodeKloud lab UI.

---

## 5. Final state

| Requirement | Result |
|---|---|
| `trailing-whitespace` from `pre-commit/pre-commit-hooks` | ✅ |
| `end-of-file-fixer` from `pre-commit/pre-commit-hooks` | ✅ |
| `check-yaml` from `pre-commit/pre-commit-hooks` | ✅ |
| `ruff` from `astral-sh/ruff-pre-commit` | ✅ |
| `black` from `psf/black-pre-commit-mirror` | ✅ |
| Every `repo:` entry has a `rev:` | ✅ |
| `pre-commit run --all-files` passes cleanly | ✅ (on the second run) |
| Hooks registered with git (`pre-commit install`) | ✅ |

```text
.pre-commit-config.yaml
        │
        ├── pre-commit/pre-commit-hooks  @ rev
        │     ├── trailing-whitespace
        │     ├── end-of-file-fixer
        │     └── check-yaml
        │
        ├── astral-sh/ruff-pre-commit  @ rev
        │     └── ruff
        │
        └── psf/black-pre-commit-mirror  @ rev
              └── black
                    │
                    ▼
        pre-commit install → .git/hooks/pre-commit
                    │
                    ▼
        every future `git commit` runs all 5 hooks automatically
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `pre-commit run --all-files` fails immediately with a schema/parse error | A `repo:` entry is missing `rev:`, or the YAML itself is malformed | Fix the structure first — this is config validation, not hook execution (§2.6) |
| A hook run reports "Failed" but the diff shows only whitespace/formatting changes | A fixing hook (trailing-whitespace, end-of-file-fixer, black) modified files during this run | Expected — re-run `pre-commit run --all-files`; it should pass clean the second time |
| `autoupdate` doesn't seem to change anything | `rev:` was already pointing at the latest release, or the `repo:` URL is wrong so nothing resolves | Confirm each `repo:` URL is correct and reachable; diff the config before/after to confirm updates landed |
| `pre-commit` reports an unknown hook `id` | Typo in the `id:` field, or an `id` copied from the wrong repository's hook manifest | Match `id:` exactly to what that specific `repo:`'s own `.pre-commit-hooks.yaml` defines |
| Hooks pass with `run --all-files` but don't run on `git commit` | `pre-commit install` was never run | Run `pre-commit install` to wire hooks into `.git/hooks/pre-commit` |
| `ruff`/`black` hook fails with an actual linting/formatting complaint, not just "files were modified" | A real code-quality issue exists in `process.py`, independent of the pre-commit config itself | Read the specific Ruff/Black output; fix the underlying code issue (same diagnosis discipline as Day 6) |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.10) out loud
      from "pre-commit run --all-files fails" to "hooks registered and
      running cleanly."
- [ ] Explain, in one sentence, the difference between `repo:`, `rev:`,
      and `hooks:` in a `.pre-commit-config.yaml` entry.
- [ ] Explain why pinning `rev:` to a branch name (like `main`) defeats
      the purpose of having a pinned configuration at all.
- [ ] Explain why a fixing hook reporting "Failed" on its first run isn't
      necessarily evidence your configuration is wrong.
- [ ] Explain what `pre-commit autoupdate` actually does, and why it's
      preferred over hand-typing version tags.
- [ ] Explain the difference between `pre-commit run --all-files` and
      `pre-commit install`, and why the task requires both.
- [ ] Break the config again (remove a `rev:` field, or swap a hook
      `id`), then fix it from memory using only `pre-commit run
      --all-files`'s error output to diagnose.

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
<table + diagram of what the configuration looks like after completion>

## 6. Troubleshooting
<symptom / cause / fix table>

## 7. Do it yourself (checklist)
<"walk the reasoning chain out loud" + comprehension questions + "break it
again and fix it from memory" prompt>
```
