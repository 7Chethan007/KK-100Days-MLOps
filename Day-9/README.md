# Day 9 — Fix a Broken Cookiecutter Template for ML Projects

A KodeKloud "100 MLOps" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive the fix
yourself*.

---

## 1. Scenario

The xFusionCorp ML platform team maintains a Cookiecutter template for
generating standardized ML projects, at `/root/code/mlops-template/`.
The template is broken. Task:

1. Fix the Cookiecutter configuration (`cookiecutter.json`).
2. Fix the Jinja templating logic (broken variable name, broken
   comparison operator).
3. Support three ML frameworks: `sklearn`, `pytorch`, `tensorflow`.
4. Generate a project at `/root/code/churn-model/` with:
   ```text
   project_name   = churn-model
   author         = xFusionCorp
   python_version = 3.11
   ml_framework   = sklearn
   ```
5. Verify the generated project's structure and contents match
   requirements — not just that the generator command exits cleanly.

---

## 2. Reasoning model — how to *derive* the fix, not memorize it

### 2.1 A template is a function; think of it that way from the start

```text
generate_project(project_name, author, python_version, ml_framework)
        │
        ▼
  /root/code/churn-model/
```

```text
INPUT (cookiecutter.json's variables, filled in per-run)
   │
   ▼
TEMPLATE (Jinja-templated file/directory structure)
   │
   ▼
GENERATED ARTIFACT (a real, concrete project)
```

The same shape as any configuration-driven generation tool —
Terraform variables → config → infrastructure; here, Cookiecutter
variables → Jinja template → project files. Two files can look
identical in the template and produce *different* generated projects
purely based on which inputs were supplied — worth holding this
separation clearly before touching any Jinja syntax.

### 2.2 Inspect the template's structure before touching anything

```bash
cd /root/code/mlops-template
find . -maxdepth 4 -print
```
```text
.
├── {{cookiecutter.project_name}}
│   ├── models/
│   ├── requirements.txt
│   ├── src/
│   ├── data/
│   ├── README.md
│   └── tests/
└── cookiecutter.json
```

`{{cookiecutter.project_name}}` as a literal directory name isn't a typo
or an error — it's a **template expression** Cookiecutter substitutes at
generation time. If `project_name = churn-model`, that directory
literally becomes `churn-model/` in the output. Recognizing this before
diagnosing anything else prevents mistaking normal template syntax for a
bug.

### 2.3 `cookiecutter.json` — the input contract the template depends on

```json
{
    "project_name": "my-ml-project",
    "author": "xFusionCorp",
    "python_version": "3.11"
}
```

This file declares every variable the template is allowed to reference —
`ml_framework` was entirely missing from the original, meaning no
template file could reference it at all, regardless of how correct the
Jinja logic elsewhere might be. The fix has to add it *as a genuine
option list*, not just a string:

```json
{
    "project_name": "my-ml-project",
    "author": "xFusionCorp",
    "python_version": "3.11",
    "ml_framework": ["sklearn", "pytorch", "tensorflow"]
}
```

A JSON list here is Cookiecutter's convention for "prompt the user to
pick one of these" — the first entry becomes the default when
`--no-input` is used (§2.7). This is a prerequisite fix: nothing
downstream that references `cookiecutter.ml_framework` can work until
this variable exists in the input contract at all.

### 2.4 Variable substitution — `{{ }}` is interpolation, not a comment or a typo

```jinja
# {{cookiecutter.project_name}}

Created by {{ cookiecutter.author }}.
```

```text
{{ expression }}   →  evaluate the expression, insert its value as text
```

Given `project_name = churn-model` and `author = xFusionCorp`, this
renders to exactly:

```text
# churn-model

Created by xFusionCorp.
```

Any variable referenced inside `{{ }}` must exist under the
`cookiecutter.` namespace, spelled and cased **exactly** as declared in
`cookiecutter.json` — which is precisely where the next bug lives.

### 2.5 The case-sensitivity bug — `Author` vs. `author`

```text
Created by {{ cookiecutter.Author }}.     ← template references "Author"
```
```json
"author": "xFusionCorp"                   ← cookiecutter.json declares "author"
```

```text
author  ≠  Author
```

Jinja/Python variable/attribute names are case-sensitive — `Author`
simply doesn't exist as an attribute on the rendering context, only
`author` does. This produces either a rendering error or an empty/
undefined substitution, depending on Jinja's configured undefined
behavior — either way, not what the template intends. The fix targets
the template file, not the config, since `cookiecutter.json`'s `author`
key is already correctly named:

```bash
sed -i 's/cookiecutter\.Author/cookiecutter.author/' \
  '{{cookiecutter.project_name}}/README.md'
```

Worth internalizing as a category: **case mismatches between a
declared variable and its reference are a silent, easy-to-miss class of
bug** — the fix is always to make the *reference* match the
*declaration*, and confirming which one is authoritative (here,
`cookiecutter.json`) before editing either.

### 2.6 `=` vs. `==` — assignment syntax leaking into a comparison context

```jinja
{% if cookiecutter.ml_framework = 'sklearn' %}     ← WRONG: assignment operator
{% if cookiecutter.ml_framework == 'sklearn' %}    ← CORRECT: equality comparison
```

```text
=   → assignment          ("set this to that")
==  → equality comparison ("is this equal to that?")
```

This is the exact same operator distinction that trips people up in
Python, JavaScript, C, and most C-family languages — Jinja's expression
syntax is deliberately Python-like, and `{% if %}` expects a boolean
expression, not an assignment statement. A single missing `=` character
is enough to break the entire conditional block, often with a parser
error rather than silently-wrong output — which is actually the *easier*
failure mode to diagnose, since Jinja refuses to render at all rather
than rendering something subtly wrong.

### 2.7 `if` / `elif` / `endif` — mutually exclusive branches, explicitly closed

```jinja
{% if cookiecutter.ml_framework == 'sklearn' %}
scikit-learn
{% elif cookiecutter.ml_framework == 'pytorch' %}
torch
{% elif cookiecutter.ml_framework == 'tensorflow' %}
tensorflow
{% endif %}
```

Using `elif` (not three independent `if` blocks) guarantees **exactly
one** branch renders — three separate `if` statements would each be
evaluated independently, and if the conditions were ever written
sloppily (or the variable held an unexpected value), you could end up
with `scikit-learn`, `torch`, *and* `tensorflow` all appearing in the
same generated `requirements.txt`, which is clearly wrong for a project
selecting one specific framework. `{% endif %}` is not optional
punctuation — Jinja's block-based control flow needs an explicit closing
tag the same way Python needs consistent indentation or C needs matching
braces; an unclosed `{% if %}` is a template syntax error, not just an
untidy file.

### 2.8 Smoke-test the template mechanism before generating the real artifact

```bash
cookiecutter --no-input /root/code/mlops-template
```

`--no-input` uses every variable's declared default rather than
prompting interactively — useful specifically for a fast, scriptable
sanity check that the template *parses and renders at all*, before
supplying the real, specific values the actual task requires:

```text
JSON parses           (cookiecutter.json is valid)
        ↓
Cookiecutter loads the template
        ↓
Jinja renders every {{ }} and {% %} without error
        ↓
A project is actually generated on disk
```

Inspecting this throwaway project's `requirements.txt` (`scikit-learn` —
the list's first/default entry) and `README.md` (`Created by
xFusionCorp.`, correctly substituted post-fix) confirms both variable
substitution and conditional rendering work *before* investing in
generating and verifying the actual required `churn-model` project.

### 2.9 Generate the actual required artifact with real values

```bash
cd /root/code
cookiecutter /root/code/mlops-template
```

Interactively supplying `project_name=churn-model`,
`author=xFusionCorp`, `python_version=3.11`, and selecting `sklearn`
from the framework list produces `/root/code/churn-model/` — the actual
deliverable, distinct from the smoke-test project generated in §2.8.

### 2.10 Verify properties, not appearance — the actual right way to check a generated artifact

```bash
test -f /root/code/churn-model/README.md && \
test -f /root/code/churn-model/requirements.txt && \
echo "Required files: PASS"

for d in data models src tests; do
  test -d "/root/code/churn-model/$d" || exit 1
done
echo "Required directories: PASS"

grep -Fxq "scikit-learn" /root/code/churn-model/requirements.txt && \
grep -Fq "churn-model" /root/code/churn-model/README.md && \
grep -Fq "xFusionCorp" /root/code/churn-model/README.md && \
echo "Generated content: PASS"
```

"Looks right" (eyeballing a directory listing) is a weak verification
compared to explicit, scriptable assertions about specific required
properties — file existence, directory existence, and exact expected
content strings. This is precisely the shape of an automated CI check,
and building the habit of verifying *this* way, even manually, is what
this section is actually teaching — the same discipline as boundary-
value testing in Day 7's `predict()` tests, or the exact-content
`.dist-info` inspection of a built wheel, applied here to a scaffolded
project's structure instead of a Python package.

### 2.11 Template vs. generated artifact — never confuse editing one for editing the other

```text
Template (/root/code/mlops-template/)     Generated artifact (/root/code/churn-model/)
        │                                              │
contains {{ }}/{% %} EXPRESSIONS               contains CONCRETE, rendered values
   NOT a real, runnable project                  an actual real project
        │                                              │
   edit THIS to fix a bug                    inspect THIS to verify the fix worked
```

Editing a file inside the *generated* `churn-model/` project to "fix"
something is treating a symptom — the next time anyone generates a new
project from this template, the same bug reappears, because the actual
defect lives in the *template*, not any one of its outputs. Every fix in
this lab targets files under `mlops-template/`; files under
`churn-model/` are only ever inspected, never edited.

### 2.12 The compressed reasoning chain

```text
Requirement (fixed template; supports 3 frameworks; churn-model generated + verified)
   → Inspect template structure                    → {{cookiecutter.project_name}} is template syntax, not a bug
   → Inspect cookiecutter.json                       → ml_framework variable entirely missing
   → Add ml_framework as a JSON list (3 options)
   → Inspect README.md template                      → cookiecutter.Author — case mismatch vs. declared "author"
   → sed-fix the reference to cookiecutter.author
   → Inspect requirements.txt template                → `=` used where `==` (equality) was needed
   → Fix to == ; confirm if/elif/elif/endif structure is complete
   → Smoke test: cookiecutter --no-input               → confirms JSON + Jinja render without error
   → Inspect smoke-test output                         → default framework's dependency + author substituted correctly
   → Generate the REAL project: cookiecutter (interactive, real values)
   → Verify PROPERTIES, not appearance: file/dir existence + exact content greps
   → Confirm every fix targeted the TEMPLATE, never the generated output
```

---

## 3. Concepts (reference)

### 3.1 Cookiecutter as project scaffolding
A tool that generates a new project's file/directory structure from a
reusable template plus a set of input variables — the goal is
consistent project starting points across a team or organization,
instead of every engineer hand-assembling the same boilerplate.

### 3.2 `cookiecutter.json` as the input contract
Declares every variable a template's files are allowed to reference.
Anything referenced in a Jinja expression but absent from this file has
no value to substitute — always the first thing to check when a
template variable "doesn't work."

### 3.3 Jinja templating — two distinct constructs
- `{{ expression }}` — variable/expression **substitution**; evaluates
  and inserts a value as text.
- `{% statement %}` — **control flow** (`if`/`elif`/`else`/`endif`,
  loops, etc.); Jinja's actual "programming language" surface, not just
  text replacement.

### 3.4 Why case sensitivity and `=`-vs-`==` matter here specifically
Jinja's expression syntax is deliberately Python-like — the same rules
that make `Author` ≠ `author` and `=` ≠ `==` matter in Python apply
directly inside `{{ }}` and `{% %}` blocks. Treating Jinja templates as
"just text with some substitution" rather than "a small embedded
programming language" is what makes bugs like these easy to introduce
and easy to overlook.

### 3.5 `--no-input` vs. interactive generation
`--no-input` renders using each variable's default (first list entry,
or a plain default value) — fast, scriptable, ideal for smoke-testing a
template's mechanics. Interactive generation (no flag) prompts for every
variable — necessary when the task requires *specific*, non-default
values, as this lab's `churn-model`/`xFusionCorp`/`sklearn` requirement
does.

### 3.6 Verifying properties vs. verifying appearance
Explicit assertions (`test -f`, `test -d`, `grep -Fxq` for exact-line
matches, `grep -Fq` for substring matches) about specific required facts
are strictly stronger evidence than a visual scan of a directory
listing — and are the same shape of check a real CI pipeline would run
automatically.

### 3.7 Template vs. generated artifact — one more time, because it's the core distinction
The template is the *source of truth*; every generated project is a
*disposable, regenerable output*. Bugs are fixed in the template.
Generated projects are inspected, never patched as a substitute for
fixing the template itself.

---

## 4. Runbook

### 4.1 Inspect the template's directory structure
```bash
cd /root/code/mlops-template
find . -maxdepth 4 -print
```
```text
.
├── {{cookiecutter.project_name}}
│   ├── models/
│   ├── requirements.txt
│   ├── src/
│   ├── data/
│   ├── README.md
│   └── tests/
└── cookiecutter.json
```

### 4.2 Inspect and fix `cookiecutter.json`
```bash
cat cookiecutter.json
```
```json
{
    "project_name": "my-ml-project",
    "author": "xFusionCorp",
    "python_version": "3.11"
}
```
```bash
cat > cookiecutter.json <<'EOF'
{
    "project_name": "my-ml-project",
    "author": "xFusionCorp",
    "python_version": "3.11",
    "ml_framework": ["sklearn", "pytorch", "tensorflow"]
}
EOF
```

### 4.3 Fix the case-sensitivity bug in README.md
```bash
grep -n "cookiecutter.Author" '{{cookiecutter.project_name}}/README.md'
```
```text
3:Created by {{ cookiecutter.Author }}.
```
```bash
sed -i 's/cookiecutter\.Author/cookiecutter.author/' \
  '{{cookiecutter.project_name}}/README.md'
```

### 4.4 Fix the `=` vs `==` bug and confirm the if/elif/endif structure in requirements.txt
```bash
cat '{{cookiecutter.project_name}}/requirements.txt'
```
```jinja
{% if cookiecutter.ml_framework = 'sklearn' %}
scikit-learn
{% elif cookiecutter.ml_framework == 'pytorch' %}
torch
{% elif cookiecutter.ml_framework == 'tensorflow' %}
tensorflow
{% endif %}
```
```bash
cat > '{{cookiecutter.project_name}}/requirements.txt' <<'EOF'
{% if cookiecutter.ml_framework == 'sklearn' %}
scikit-learn
{% elif cookiecutter.ml_framework == 'pytorch' %}
torch
{% elif cookiecutter.ml_framework == 'tensorflow' %}
tensorflow
{% endif %}
EOF
```

### 4.5 Smoke-test the template
```bash
cd /root/code
cookiecutter --no-input /root/code/mlops-template
```
```text
my-ml-project/
├── models/
├── requirements.txt
├── src/
├── data/
├── README.md
└── tests/
```
```bash
cat my-ml-project/requirements.txt
```
```text
scikit-learn
```
```bash
cat my-ml-project/README.md
```
```text
# my-ml-project

Created by xFusionCorp.
```
Both variable substitution and conditional rendering confirmed working.
```bash
rm -rf my-ml-project
```

### 4.6 Generate the required project with real values
```bash
cookiecutter /root/code/mlops-template
```
```text
project_name [my-ml-project]: churn-model
author [xFusionCorp]: xFusionCorp
python_version [3.11]: 3.11
Select ml_framework:
1 - sklearn
2 - pytorch
3 - tensorflow
Choose from 1, 2, 3 [1]: 1
```
```bash
find /root/code/churn-model -maxdepth 2 -print
```
```text
.
├── models/
│   └── .gitkeep
├── requirements.txt
├── src/
│   └── .gitkeep
├── data/
│   └── .gitkeep
├── README.md
└── tests/
    └── .gitkeep
```

### 4.7 Inspect the generated content
```bash
cat /root/code/churn-model/requirements.txt
```
```text
scikit-learn
```
```bash
cat /root/code/churn-model/README.md
```
```text
# churn-model

Created by xFusionCorp.
```

### 4.8 Verify by property, not by eye
```bash
test -f /root/code/churn-model/README.md && \
test -f /root/code/churn-model/requirements.txt && \
echo "Required files: PASS"
```
```text
Required files: PASS
```
```bash
for d in data models src tests; do
  test -d "/root/code/churn-model/$d" || exit 1
done
echo "Required directories: PASS"
```
```text
Required directories: PASS
```
```bash
grep -Fxq "scikit-learn" /root/code/churn-model/requirements.txt && \
grep -Fq "churn-model" /root/code/churn-model/README.md && \
grep -Fq "xFusionCorp" /root/code/churn-model/README.md && \
echo "Generated content: PASS"
```
```text
Generated content: PASS
```

### 4.9 Submit
With the template fixed, all three frameworks supported, and
`churn-model` verified by property, click **Check** in the KodeKloud lab
UI.

---

## 5. Final state

| Requirement | Result |
|---|---|
| `cookiecutter.json` declares all 4 variables, `ml_framework` as a 3-option list | ✅ |
| README's author reference fixed (`author`, not `Author`) | ✅ |
| `requirements.txt`'s `if`/`elif`/`endif` uses `==`, is properly closed | ✅ |
| Template smoke-tests cleanly via `--no-input` | ✅ |
| `churn-model/` generated with `project_name`, `author`, `python_version`, `ml_framework=sklearn` | ✅ |
| Required files present (`README.md`, `requirements.txt`) | ✅ |
| Required directories present (`data`, `models`, `src`, `tests`) | ✅ |
| Generated content correct (`scikit-learn`, `churn-model`, `xFusionCorp`) | ✅ |

```text
mlops-template/  (TEMPLATE — source of truth)
   │
   ├── cookiecutter.json         (4 variables, ml_framework as a list)
   └── {{cookiecutter.project_name}}/
         ├── README.md            {{ cookiecutter.author }}
         └── requirements.txt     {% if/elif/endif ml_framework %}
                    │
                    │ cookiecutter (real values)
                    ▼
churn-model/  (GENERATED ARTIFACT — disposable, inspected only)
   ├── README.md          "Created by xFusionCorp."
   ├── requirements.txt   "scikit-learn"
   ├── data/  models/  src/  tests/
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| A referenced variable seems to have no value / renders empty or errors | Variable missing from `cookiecutter.json`, or the template's reference is cased differently than the declaration | Confirm the exact key spelling/case in `cookiecutter.json`; fix the *reference* to match the *declaration* |
| `{% if %}` block fails to parse, or renders unexpectedly | `=` used instead of `==` for equality comparison | Use `==` for comparisons; `=` is assignment, not valid inside an `if` condition |
| Generated file contains output from multiple mutually-exclusive branches (e.g. all three frameworks) | Used independent `{% if %}` blocks instead of `{% if %}`/`{% elif %}` | Restructure as a single `if`/`elif`/`elif`/`endif` chain |
| Jinja reports an unclosed block / template syntax error | Missing `{% endif %}` (or `{% endfor %}`, etc.) | Every Jinja control block must be explicitly closed |
| `cookiecutter --no-input` "succeeds" but the task still fails verification | Trusted the generator's exit code instead of inspecting the actual generated files | Always follow generation with explicit content/structure checks (§2.10) |
| A fix seems to "not take effect" on a newly generated project | The fix was applied inside a previously-*generated* project instead of the *template* | Confirm every edit targets files under the template directory, never under a generated output (§2.11) |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.12) out loud
      from "the Cookiecutter template is broken" to "churn-model
      verified by property."
- [ ] Explain, in one sentence, the difference between `{{ }}` and
      `{% %}` in Jinja.
- [ ] Explain why `cookiecutter.Author` failed even though
      `cookiecutter.json` clearly had an `author` key.
- [ ] Explain why `=` inside a Jinja `{% if %}` is a bug, using the same
      reasoning you'd use for a similar mistake in Python.
- [ ] Explain why `if`/`elif`/`elif`/`endif` is correct here instead of
      three separate `if` blocks.
- [ ] Explain why verifying with `test -f`/`grep -Fxq` is stronger
      evidence than visually checking the generated project looks right.
- [ ] Break the template again (revert one fix), regenerate, and
      diagnose it from memory using only the generated output's
      behavior — not this file.

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
