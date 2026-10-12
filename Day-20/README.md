# Day 20 — Starting an MLflow Tracking Server

A KodeKloud "100 MLOps" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive it yourself*.

Days 10-19 all tracked *data and models* with DVC against a dataset
or pipeline a single engineer runs locally. This lab stands up the
piece that makes that work *shared and comparable across people and
runs*: a tracking service other processes report into over HTTP,
rather than a local tool run against a local Git repo. The backend-
store/artifact-store split introduced here directly parallels DVC's
Git-metadata-vs-cached-content split from Day 18.

---

## 1. Scenario

Start an MLflow Tracking Server on `controlplane`, configured so the
lab proxy (a non-loopback network path) can reach it and render its
UI in a browser.

| Requirement | Configuration |
|---|---|
| Port | `5000` |
| Network binding | `0.0.0.0` |
| Backend store | SQLite at `/root/code/mlflow-backend/mlflow.db` |
| Artifact root | `/root/code/mlflow-artifacts/` |
| Host-header validation | Allow any host (`--allowed-hosts '*'`) |
| CORS | Allow any origin (`--cors-allowed-origins '*'`) |
| Process lifecycle | Background, detached from the terminal |

```text
ML engineer / training script
         │ HTTP (tracking API)
         ▼
   Lab proxy ──▶ MLflow server :5000 (bound 0.0.0.0)
                      │                    │
                      ▼                    ▼
         SQLite backend store      Artifact filesystem
         (runs/params/metrics)     (model files, plots)
```

---

## 2. Reasoning model — how to *derive* the commands, not memorize them

### 2.1 A tracking server is infrastructure, not a model-training artifact

```text
This lab produces:     a running SERVICE other processes report into
This lab does NOT:      train a model, or produce any experiment data itself
```

Every prior MLOps lab (Days 10-19) operated on data/pipelines that
already existed on disk. This one is the first to stand up a *service*
with no ML content of its own yet — success here is measured by "can a
future training run report into this," not by any metric. Keep this
distinction explicit, because it's easy to mistake "server started" for
"experiment tracked" when they're unrelated claims.

### 2.2 Check the tool's actual capabilities before constructing the command

```bash
mlflow server --help
```

Confirming `--allowed-hosts` and `--cors-allowed-origins` exist in the
installed version (3.16.1) *before* writing the full startup command
avoids discovering a missing flag only after a failed launch. This is
the same "inspect before you act" discipline as Day 19 DevOps's
`grep Listen/DocumentRoot` before editing `httpd.conf` — check the
actual current capability/state, don't assume it from memory of a
different version or a different tool.

### 2.3 SQLite can create a file, but never a missing parent directory

```bash
mkdir -p /root/code/mlflow-backend
mkdir -p /root/code/mlflow-artifacts
```

`--backend-store-uri sqlite:////root/code/mlflow-backend/mlflow.db`
only works if `mlflow-backend/` already exists — SQLite will happily
create the `.db` file itself, but it has no mandate to create
directories on your behalf. Pre-creating both required directories
eliminates one entire predictable class of startup failure before the
server is ever launched, rather than discovering it from a crash log
afterward.

### 2.4 `nohup ... &` solves a specific, narrow problem — terminal detachment, not persistence

```text
nohup mlflow server ... &
```

This combination keeps the process alive after the interactive shell
session that launched it closes, and returns control to the terminal
immediately. It does **not** make the process survive a reboot, restart
automatically on crash, or get supervised in any way — that would
require a real service manager (`systemd`, a container orchestrator).
Conflating "detached from this terminal" with "production-grade
process lifecycle" is the specific mistake this distinction guards
against.

### 2.5 `--host 0.0.0.0` is a bind address, not a URL to browse to

```text
0.0.0.0  → "accept connections on every network interface this host has"
127.0.0.1 → "accept connections ONLY via loopback" — the lab proxy can't reach this
```

The lab proxy connects over a non-loopback network path, so a server
bound to `127.0.0.1` would be invisible to it even while working
perfectly for local `curl` tests on the same machine. `0.0.0.0` is
purely about *which interfaces the server listens on* — the actual
browser-facing URL is whatever the lab proxy exposes, a separate and
unrelated piece of configuration the server itself knows nothing about.

### 2.6 Host-header validation and CORS are two different browser/network security checks

```text
Host validation:  checks the HTTP Host header → mitigates DNS rebinding
CORS:             controls whether browser JS from one origin can read another origin's response
```

Both needed to be relaxed to `*` for this lab specifically because the
lab proxy's requests arrive with a `Host` header that isn't
`localhost`, and the browser's JS is making cross-origin requests
through the proxy. These are independent mechanisms solving different
problems — passing one wouldn't have implied the other was also
satisfied, which is why both flags needed to be set and both were
separately visible in the startup log (`Allowed hosts: *. CORS
origins: *.`).

### 2.7 Backend store vs. artifact store — the same split DVC makes between Git metadata and cached content

```text
Backend store (SQLite):  structured metadata → run IDs, params, metrics, tags — needs QUERYING
Artifact store (files):   model files, plots, reports — needs FILESYSTEM/OBJECT storage
```

A relational store suits "find every run where `max_depth > 5`
ordered by F1" in a way a flat file collection doesn't; files suit
"store this 60MB model binary" in a way database rows don't. This is
structurally the same reason DVC keeps content-addressed artifact
bytes in a cache separate from small, diffable `.dvc`/`dvc.lock`
metadata in Git (Day 18 §2.1) — small structured facts and large
binary content are stored differently because they're queried and
used differently, not because of an arbitrary tooling choice.

### 2.8 Verifying four independent layers, because a shell prompt returning doesn't prove any of them

```text
pgrep -af 'mlflow server'   → the process still EXISTS (a launched command can exit seconds later)
tail /tmp/mlflow-server.log → it actually finished STARTUP (DB tables created, bound correctly)
curl -I http://127.0.0.1:5000/ → the HTTP endpoint actually RESPONDS, locally
browser via lab proxy       → the full chain works from the ACTUAL client path
```

Each check answers a different question and none subsumes another —
a process can exist but have failed its DB initialization; a log can
show successful startup while something else later crashes the
listener; `curl` succeeding locally says nothing about whether the
*proxy* path (the actual client-facing route) also works. This is the
same "verify the decisive end-to-end behavior, plus the layers
underneath it" principle as Day 19 DevOps's Nginx+PHP-FPM chain —
just with the shell-prompt-returning step added as the first, weakest
layer that checks nothing on its own (§2.4's "a command can return
control to the shell even if the child process exits shortly after").

### 2.9 Starting the server creates the service; nothing reports into it automatically

```python
mlflow.set_tracking_uri("http://<tracking-server-host>:5000")
mlflow.set_experiment("fraud-detection")
with mlflow.start_run():
    mlflow.log_param(...)
    mlflow.log_metric(...)
```

A training script has to explicitly point itself at the tracking
server's URI and call `log_param`/`log_metric`/etc. — the server
doesn't intercept or auto-discover training runs happening elsewhere.
Using `127.0.0.1` here would be wrong whenever the training process
runs on a different machine or container than the tracking server
itself; the URI has to be whatever address is actually reachable from
wherever the client code runs, the same host-vs-loopback distinction
as §2.5 but from the client's side this time.

### 2.10 The compressed reasoning chain

```text
Requirement: shared MLflow tracking service, reachable via the lab proxy
   → mlflow server --help → confirm the installed version supports the needed flags (§2.2)
   → mkdir -p both backend-store and artifact-root directories FIRST (§2.3)
   → single launch command specifying every requirement at once:
       --host 0.0.0.0 (proxy needs non-loopback, §2.5)
       --backend-store-uri sqlite:///...  (structured metadata, §2.7)
       --default-artifact-root ...         (file storage, §2.7)
       --allowed-hosts '*' --cors-allowed-origins '*'  (two SEPARATE checks, §2.6)
   → nohup ... & for terminal detachment ONLY, not reboot persistence (§2.4)
   → verify FOUR independent layers: pgrep → log tail → curl → browser via proxy (§2.8)
   → (separately, later) a training script must set_tracking_uri + log_* explicitly (§2.9)
```

---

## 3. Concepts (reference)

### 3.1 Experiment tracking vs. model training
A tracking server (MLflow) records parameters, metrics, and artifacts
that training jobs choose to log. It performs no training itself and
captures nothing automatically unless a client explicitly logs it.

### 3.2 Backend store vs. artifact store
The backend store holds structured, queryable metadata (run IDs,
parameters, metrics, tags); the artifact store holds files (models,
plots, reports). Separating them lets each be backed by storage suited
to its actual access pattern, and lets either be swapped later (e.g.
SQLite → PostgreSQL, local filesystem → object storage) without
changing the tracking workflow itself.

### 3.3 Network bind address vs. client-facing URL
`--host` controls which network interfaces the server process listens
on. It says nothing about what URL a browser or client actually
connects to — that's determined by whatever sits in front of the
server (a proxy, a load balancer, DNS).

### 3.4 Host-header validation vs. CORS
Host validation guards against a request's `Host` header being used
maliciously (DNS rebinding); CORS governs whether a browser permits
cross-origin JavaScript to read a response. Both are enforced
independently by the server, and both needed relaxing here for
entirely separate reasons tied to how the lab proxy routes traffic.

### 3.5 Process detachment vs. service supervision
`nohup`/`&` remove a process's dependency on the launching shell
session remaining open. They provide no crash recovery, no restart
policy, and no reboot persistence — that requires a dedicated service
manager or orchestrator.

### 3.6 Independent, layered verification
Checking that a launch command returned control to the shell, that
the process still exists, that its logs show successful
initialization, that it answers locally over HTTP, and that it's
reachable via the actual client path are five different claims — none
proves any of the others.

---

## 4. Runbook

### 4.1 Confirm the execution environment
```bash
whoami
mlflow --version
mlflow server --help
```
```text
root
mlflow, version 3.16.1
```
Confirmed `--allowed-hosts`/`--cors-allowed-origins` are supported by
this installed version; running as `root` avoids permission issues
under `/root/code/`.

### 4.2 Create the storage directories
```bash
mkdir -p /root/code/mlflow-backend
mkdir -p /root/code/mlflow-artifacts
ls -ld /root/code/mlflow-backend /root/code/mlflow-artifacts
```

### 4.3 Start the server as a detached background process
```bash
nohup mlflow server \
  --host 0.0.0.0 \
  --port 5000 \
  --backend-store-uri sqlite:////root/code/mlflow-backend/mlflow.db \
  --default-artifact-root /root/code/mlflow-artifacts/ \
  --allowed-hosts '*' \
  --cors-allowed-origins '*' \
  > /tmp/mlflow-server.log 2>&1 &
```

### 4.4 Verify the running process
```bash
pgrep -af 'mlflow server'
```
```text
3307 /usr/bin/python3 /usr/local/bin/mlflow server ...
```

### 4.5 Inspect the startup log
```bash
tail -n 40 /tmp/mlflow-server.log
```
```text
Creating initial MLflow database tables...
Updating database tables
[MLflow] Security middleware enabled. Allowed hosts: *. CORS origins: *.
Uvicorn running on http://0.0.0.0:5000
Application startup complete.
```

### 4.6 Verify HTTP locally and via the browser
```bash
curl -I http://127.0.0.1:5000/
```
```text
HTTP/1.1 200 OK
content-type: text/html; charset=utf-8
access-control-allow-origin: *
```
Then opened the lab's MLflow UI button — the Home page (version
`3.16.1`, Default experiment) loaded through the proxy, confirming the
full client-facing path works end to end.

### 4.7 (Reference only) how a training script would report into this server
```python
import mlflow
mlflow.set_tracking_uri("http://<tracking-server-host>:5000")
mlflow.set_experiment("fraud-detection")
with mlflow.start_run():
    mlflow.log_param("learning_rate", 0.01)
    mlflow.log_param("batch_size", 32)
    mlflow.log_metric("validation_f1", 0.87)
```
Not executed in this lab — no training run occurred. Shown only to
illustrate what the deployed service is for.

---

## 5. Final state

| Requirement | Result |
|---|---|
| MLflow version/flags confirmed before launch | ✅ |
| Backend/artifact directories created | ✅ |
| Server process running (`pgrep`) | ✅ |
| SQLite tables initialized (log confirms) | ✅ |
| Bound to `0.0.0.0:5000` | ✅ |
| Host/CORS wildcards active | ✅ |
| Local HTTP responds `200 OK` | ✅ |
| Browser UI reachable via lab proxy | ✅ |

```text
controlplane
 └── mlflow server (pid 3307), nohup-detached, 0.0.0.0:5000
       ├── backend-store: sqlite:////root/code/mlflow-backend/mlflow.db
       └── artifact-root: /root/code/mlflow-artifacts/
            reachable via: lab proxy → browser UI (v3.16.1)
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Server exits right after starting | Invalid option, missing directory, or DB error | `tail -n 40 /tmp/mlflow-server.log` |
| Connection refused | Process exited, or not actually listening | `pgrep -af 'mlflow server'` + check logs |
| Local `curl` works but the lab proxy UI doesn't | Proxy routing, or host/CORS validation rejecting the proxy's headers | Re-check `--allowed-hosts`/`--cors-allowed-origins`, inspect browser network errors |
| UI loads but shows no meaningful experiments | Server running correctly — no training runs have been logged yet | Expected; log a run via `mlflow.start_run()` to populate it |
| Backend `.db` file missing | Initialization didn't complete, or path differs from configured URI | `ls -lh /root/code/mlflow-backend/mlflow.db` |
| Artifact upload fails from a training script | Wrong artifact URI, storage path, or permissions | Inspect the experiment's artifact URI and server logs |
| Command appears to "hang" or errors on an unrelated filename | Multiple commands pasted together and misinterpreted as one | Run commands one at a time, inspect each output before continuing |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.10) out loud
      from "`mlflow server --help`" to "verified via browser through
      the proxy."
- [ ] Explain, in one sentence, why `nohup ... &` is not equivalent to
      reboot persistence.
- [ ] Explain why host-header validation and CORS are two separate
      checks, both of which needed relaxing, rather than one setting
      covering both.
- [ ] Explain why the backend store and artifact store are configured
      as two separate locations instead of one.
- [ ] Explain why `pgrep` confirming the process exists doesn't by
      itself prove the server started successfully.
- [ ] From memory, write (don't run, unless you have a safe
      environment) the three-line Python snippet that would log a
      parameter and a metric into this server from a separate
      training script, substituting the correct tracking URI.
- [ ] Start a second MLflow server on a different port with a
      different backend/artifact path, verify it independently through
      all four layers (§2.8), then stop it cleanly.

---

## 8. Document template for future lab days

Use this skeleton for `day-N/README.md`:

```markdown
# Day N — <task title>

## 1. Scenario
<what was asked, which account/region/resources, constraints, credentials
if relevant>

## 2. Reasoning model — how to derive the commands
<walk the requirement down to a subsystem/resource hierarchy, step by step,
in the order you'd actually discover it: "what resource, is it global or
region-scoped, what existing state constrains my choice, what CLI
service+operation performs the read (exact lookup vs. filtered search),
what performs the write, how do I verify the SPECIFIC field the
requirement names." End with a compressed step-chain.>

## 3. Concepts (reference)
<one subsection per concept the task exercises — explain WHY, not just WHAT>

## 4. Runbook
<copy-pasteable commands in the order run, WITH the actual intermediate
output/results captured inline as code blocks, not just the commands>

## 5. Final state
<table + diagram of what the infrastructure looks like after completion>

## 6. Troubleshooting
<symptom / cause / fix table>

## 7. Do it yourself (checklist)
<"walk the reasoning chain out loud" + comprehension questions +
"redo without copy-pasting, recompute derived values" prompt>
```
