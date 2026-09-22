# Day 2 — Fix a Broken JupyterLab Server Configuration

A KodeKloud "100 MLOps" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive it yourself*.

---

## 1. Scenario

xFusionCorp's data science team has a JupyterLab server that won't start
correctly. Task: fix `/root/code/jupyter_lab_config.py` so JupyterLab
starts and is reachable, then actually start it and confirm it's usable.

Broken config found:

```python
c.IdentityProvider.token = ''
c.ServerApp.disable_check_xsrf = True
c.ServerApp.root_dir = '/root/wrong-path'
c.ServerApp.port = 8000
c.ServerApp.ip = '1.1.1.1'
```

Requirements:
- root/notebook directory → `/root/notebooks`
- port → `8888`
- IP → `0.0.0.0`
- JupyterLab install to use: `/root/code/ml-env/bin/jupyter-lab` (v4.6.4)

---

## 2. Reasoning model — how to *derive* the fix, not memorize it

### 2.1 The system, mapped

```text
Linux machine
│
├── Python environment
│   └── /root/code/ml-env/            ← which jupyter binary actually runs
│
├── Jupyter configuration
│   └── /root/code/jupyter_lab_config.py   ← desired state, as code
│
└── JupyterLab server (a process)
    ├── network: IP + port it binds to
    └── filesystem: root_dir it serves
```

The task is a generic pattern: **make the application config match the
required infrastructure, then verify the running process actually reaches
its "serving" state** — not just "the command didn't crash."

### 2.2 Inspect before touching anything

```bash
cat /root/code/jupyter_lab_config.py
```

A config file only tells you *intent*; you can't fix a mismatch you
haven't read yet. Diffing what's there against what's required:

```text
              current                required
root_dir      /root/wrong-path   →   /root/notebooks
port          8000                →   8888
ip            1.1.1.1             →   0.0.0.0
```

### 2.3 A config path referencing a directory doesn't create it

```bash
ls -ld /root/notebooks/
# No such file or directory
```

`root_dir = '/root/notebooks'` in the config is just an instruction —
Jupyter will try to serve from that path, but nothing about writing that
line makes the directory exist. Same failure mode you'll hit with S3
buckets, DB schemas, mount points: **config says where; filesystem/infra
must independently already have it, or you must provision it.**

```text
Application config  →  "use this directory"  →  filesystem  →  exists? (check separately)
```

### 2.4 Confirm which binary you're actually going to run

```bash
/root/code/ml-env/bin/jupyter-lab --version
```

A machine can have multiple `jupyter-lab` installs (`/usr/bin/...`, a venv,
a conda env, a user's `~/.local/bin/...`) each with different package
versions. Explicitly pointing at the one under `/root/code/ml-env/` avoids
the classic "works in my shell, not in the app's actual environment" bug —
you want to know, concretely, which environment's dependency set you're
testing against before you start debugging failures.

### 2.5 Reproduce the failure before changing anything

```bash
/root/code/ml-env/bin/jupyter-lab --config=/root/code/jupyter_lab_config.py
```

Deliberately running the *broken* config first is the useful move here.
The debugging loop that generalizes to almost any broken-service task:

```text
Observe  →  Reproduce  →  Understand the failure  →  Change one thing  →  Test again
```

— not "see an error, change five things, hope." Changing one variable at a
time is what lets you attribute each error message to its actual cause.

### 2.6 Reading the first failure: socket binding

```text
OSError: [Errno 99] Cannot assign requested address
```

caused by:

```python
c.ServerApp.ip = '1.1.1.1'
```

Starting an HTTP server is conceptually:

```text
create socket → bind(ip, port) → listen → accept connections
```

`bind()` only succeeds on an IP address that actually belongs to a network
interface on *this* machine. `1.1.1.1` is a public DNS address (Cloudflare)
that isn't assigned to any local interface here — so the OS refuses the
bind. The fix isn't "try a different random IP," it's understanding that
bind targets must be local-interface addresses (or the wildcard, §2.7).

### 2.7 Why `0.0.0.0` is the right choice, not just "the required value"

Three categories of bind address, in order of how much they expose:

```text
127.0.0.1        → only this machine can connect (loopback only)
<specific IP>     → only reachable via that one interface
0.0.0.0           → listen on every IPv4 interface this host has
```

Inside a container/VM/remote lab box, you almost always need `0.0.0.0` —
if you bind to `127.0.0.1`, only processes *inside* that same
container/VM can reach it; anything connecting from outside (your browser,
the lab's port-forward/proxy) hits a closed door. `0.0.0.0` is "accept
connections arriving on any interface," which is what makes the server
externally reachable at all.

### 2.8 Port is a second coordinate, not a random number

```text
IP   = which machine/interface
Port = which application/service on that machine
```

`8888` isn't magic — it's just "the port Jupyter conventionally uses" (same
way SSH defaults to 22, Postgres to 5432, Redis to 6379). The task
specifies it explicitly because *something else* (a lab port-forward, a
firewall rule, a UI proxy) is already configured to expect Jupyter on
`8888` — changing the port without changing whatever's expecting it there
would just move the mismatch.

### 2.9 `root_dir` — matching "where the app looks" to "where the data is"

```python
c.ServerApp.root_dir = '/root/notebooks'
```

tells Jupyter "expose this filesystem path as the workspace root." Same
category of setting as a web server's document root, a database's data
directory, or an app's working directory — the app doesn't create it for
you, so pair the config edit with actually provisioning the path:

```bash
mkdir -p /root/notebooks
```

`-p` = create any missing parent directories, and don't error if the
target already exists — the idempotent way to say "ensure this path
exists," safe to re-run.

### 2.10 The root-user warning is a security gate, not a bug

```text
Running as root is not recommended. Use --allow-root to bypass.
```

Root can read/modify/execute/destroy anything on the system. Jupyter lets
you execute arbitrary code from a notebook — if that process runs as root,
anyone with notebook access effectively has root. Jupyter refuses to start
as root by default specifically to make you *opt in* consciously:

```bash
jupyter lab --config=/root/code/jupyter_lab_config.py --allow-root --no-browser &
```

Fine for a disposable lab sandbox; in a real deployment the fix is "run
this as a dedicated non-root user," not "always pass `--allow-root`."

### 2.11 Activating the venv changes *what `jupyter` resolves to*, not just aesthetics

```bash
source /root/code/ml-env/bin/activate
```

Before: your shell's `PATH` resolves `python`/`pip`/`jupyter` to whatever
system-level install comes first. After: those names resolve to the
binaries inside `/root/code/ml-env/` first. Verify it actually took effect:

```bash
which python
which jupyter
# both paths should be under /root/code/ml-env/
```

This is one of the highest-value debugging habits for any Python-based
MLOps system: when something "isn't installed" or "is the wrong version,"
check which environment is actually active before assuming the package is
missing.

### 2.12 Don't grep logs for the word "error" — look for the success state

```text
[W ...] The module 'notebook' could not be found  ← looks scary, has a traceback
...
[I ...] jupyterlab | extension was successfully loaded
[I ...] Jupyter Server 2.21.1 is running at: ...
```

A traceback-shaped warning isn't automatically fatal. The reliable signal
is whether the process reaches its documented "ready" state — here,
`Serving notebooks from local directory: /root/notebooks` plus `... is
running at: ...`. Read logs for **did it reach the success state**, not
**does the word "error" appear anywhere**.

### 2.13 The compressed reasoning chain

```text
Requirement (Jupyter reachable at 0.0.0.0:8888, serving /root/notebooks)
   → Inspect current config (cat)                      → found wrong ip/port/root_dir
   → Confirm which jupyter binary runs (--version)      → /root/code/ml-env/bin/jupyter-lab
   → Reproduce with broken config                       → OSError: Cannot assign requested address
   → Diagnose: bind target must be a local interface     → fix ip → 0.0.0.0
   → Re-run                                              → root_dir doesn't exist
   → mkdir -p /root/notebooks; fix root_dir in config
   → Re-run                                              → root-user refusal
   → --allow-root (lab context; not production-safe)
   → Activate venv so `jupyter` resolves inside ml-env
   → Start in background, verify "Server ... is running" (success state, not absence of warnings)
   → Open the UI, confirm it actually works end to end
```

---

## 3. Concepts (reference)

### 3.1 Jupyter config layers
`c.ServerApp.*` (and, in newer Jupyter, `c.IdentityProvider.token` for auth
token) are `traitlets`-based config — a Python file that sets attributes on
config objects, evaluated at server startup. Key settings here:
- `token` / `password` — authentication for the web UI; empty string means
  no auth prompt (fine for an isolated lab, not for anything reachable
  publicly).
- `disable_check_xsrf` — disables CSRF protection; again, a lab-only
  relaxation.
- `root_dir` — filesystem path exposed as the notebook workspace.
- `ip` / `port` — network bind address for the HTTP server.

### 3.2 Socket binding vs. routing
Binding (`ip:port`) only concerns *this host* accepting connections on that
address/port. It says nothing about whether traffic can actually reach the
host from outside (firewalls, NAT, container port-mapping, lab
port-forwarding are separate concerns layered on top).

### 3.3 `0.0.0.0` vs `127.0.0.1` vs a specific IP
Already covered in §2.7 — repeat here as the one networking fact worth
over-learning: `0.0.0.0` means "all local interfaces," not "the internet."
It doesn't make the service public by itself; it just stops restricting
which *local* interface can be used to reach it.

### 3.4 Virtual environments and `PATH` resolution
A Python venv (`ml-env` here) is a directory with its own `bin/python`,
`bin/pip`, and any packages installed into it (`bin/jupyter-lab`, etc.).
`source .../activate` prepends that `bin/` directory to `PATH` for the
current shell session only — it doesn't change anything system-wide or
persist to new shells.

### 3.5 `mkdir -p`
`-p`: create intermediate directories as needed, and exit successfully
even if the target already exists — makes the command safe to re-run
(idempotent), which matters for scripts/runbooks you'll execute more than
once.

### 3.6 `--allow-root`
An explicit opt-in flag Jupyter requires before it will run its process
(and therefore any notebook-executed code) as the root user — a deliberate
friction point, not a bug to silence reflexively.

---

## 4. Runbook

### 4.1 Inspect current state
```bash
cat /root/code/jupyter_lab_config.py
ls -ld /root/notebooks/                              # confirm missing
/root/code/ml-env/bin/jupyter-lab --version           # confirm which install
```

### 4.2 Reproduce the failure (optional but recommended)
```bash
/root/code/ml-env/bin/jupyter-lab --config=/root/code/jupyter_lab_config.py
# OSError: [Errno 99] Cannot assign requested address   (from ip = 1.1.1.1)
```

### 4.3 Fix the config
Edit `/root/code/jupyter_lab_config.py` to:
```python
# Jupyter configuration file for the xFusionCorp Industries data science team

# --- xFusionCorp team overrides (review before starting the server) ---
c.ServerApp.token = ''
c.ServerApp.password = ''
c.ServerApp.disable_check_xsrf = True
c.ServerApp.notebook_dir = '/root/notebooks/'
c.ServerApp.port = 8888
c.ServerApp.ip = '0.0.0.0'
```

### 4.4 Create the required directory
```bash
mkdir -p /root/notebooks
```

### 4.5 Activate the correct environment
```bash
source /root/code/ml-env/bin/activate
which python; which jupyter     # sanity-check both resolve under ml-env
```

### 4.6 Start JupyterLab
```bash
jupyter lab --config=/root/code/jupyter_lab_config.py --allow-root --no-browser &
```

### 4.7 Verify — look for the success state, not absence of warnings
```text
[I ...] Serving notebooks from local directory: /root/notebooks
[I ...] Jupyter Server 2.21.1 is running at: http://0.0.0.0:8888/lab?token=...
```

### 4.8 Open the UI
Use the lab's top-right "Jupyter" / port-access option to open the web UI
and confirm the notebook workspace loads and is usable.

---

## 5. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `OSError: Cannot assign requested address` | `ip` set to an address not owned by any local interface | Set `c.ServerApp.ip = '0.0.0.0'` |
| `ls: cannot access '/root/notebooks/': No such file or directory` | Config points at a path that was never created | `mkdir -p /root/notebooks` |
| `Running as root is not recommended` and exits | Jupyter refuses to start as root by default | Add `--allow-root` (lab-only; use a non-root user in production) |
| `jupyter` / `python` resolve to unexpected versions or "module not found" | venv not activated, wrong environment active | `source /root/code/ml-env/bin/activate`; `which jupyter` to confirm |
| Scary traceback in logs but server seems to keep running | Non-fatal warning (e.g. optional module not found) | Check for `Jupyter Server ... is running at: ...` before concluding failure |
| UI unreachable even though server says "running" | Port mismatch with lab's port-forward/proxy expectation | Confirm port matches what the lab UI/proxy expects (`8888` here) |

---

## 6. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.13) out loud from
      "JupyterLab won't start" to "verified working in the browser."
- [ ] Explain, in one sentence, why binding to `1.1.1.1` fails on a local
      machine but binding to `0.0.0.0` succeeds.
- [ ] Explain the difference between "the config says root_dir = X" and
      "X exists on disk," and why they're two separate checks.
- [ ] Explain why `--allow-root` is a deliberate opt-in rather than a flag
      you should reach for by default.
- [ ] Break the config again (revert `ip`/`port`/`notebook_dir`), then fix
      it from memory using only the reasoning chain, not this file.
- [ ] Verify success by reading the log for the "is running at" line, not
      by assuming "no crash = working."

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
reproduce the failure → diagnose root cause (one change at a time) → fix →
re-verify → repeat until the success state, not just "no error," is
observed. End with a compressed arrow-chain version.>

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
