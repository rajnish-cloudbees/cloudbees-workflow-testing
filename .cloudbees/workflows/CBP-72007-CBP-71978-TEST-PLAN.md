# Local Tilt test plan — CBP-72007 + CBP-71978 PRs

Three PRs, three workflows. Each section: what to build, how to trigger, what to observe, and the
before/after that proves the fix.

| Workflow | PR under test | Repo to rebuild |
|---|---|---|
| `cbp-72007-skip-result-scope` | dsl-engine-cli#982 | `dsl-engine-cli-compile` |
| `cbp-72007-step-status` | cloudbees-automation-controller#552 | controller compile + deploy |
| `cbp-71978-skip-scm-status` | scm-providers#105 | scm-service (via go.work on scm-providers) |

Cluster context: `k3d-guide-rails`. A run executes in a per-run namespace
`event-<workflowname>-<hash>`, created on trigger and GC'd minutes after completion — inspect
promptly. Trigger each workflow from the component's Runs view (workflow_dispatch), except
`cbp-71978-skip-scm-status` which is `pull_request` only (open/update a PR).

---

## 1. dsl-engine-cli#982 — `cbp-72007-skip-result-scope`

The engine runs per-trigger as a meta-pipeline Task, so a rebuild takes effect on the next run
with no pod rollout.

**Build:** on the dsl-engine-cli PR branch, trigger the engine rebuild:
```
curl -s -X POST localhost:10350/api/trigger -H 'Content-Type: application/json' \
  -d '{"manifest_names":["dsl-engine-cli-compile"],"build_reason":16}'
```
Wait for `status.updateStatus=ok` via `curl -s localhost:10350/api/view`.

**Trigger** `cbp-72007-skip-result-scope`, then grab the run namespace:
```
kubectl get ns | grep event-cbp72007skipresultscope
NS=event-cbp72007skipresultscope-<hash>
```

**Observe — the structural change (primary).** List the main job Tasks and check `.spec.results`:
```
kubectl get task -n "$NS" -o custom-columns=\
'NAME:.metadata.name,RESULTS:.spec.results[*].name' | grep '^job-'
```
Expected:

| Job | `.spec.results` contains `skip`? | Why |
|---|---|---|
| `job-if-only-no-dep-*` | **no** | has `if:` but nothing reads it — #982 drops it |
| `job-needs-only-no-dep-*` | **no** | has `needs:` but nothing reads it |
| `job-standalone-*` | **no** | baseline, unchanged |
| `job-anchor-ok-*` | **yes** | needed by `dependent-of-ok` |
| `job-anchor-skip-*` | **yes** | needed by `dependent-of-skip` |
| `job-needed-by-only-*` | **yes** | needed by `dependent-of-needed-by` |

**Observe — behavior unchanged (run graph).** In the run-detail graph:
- `if-only-no-dep` → SKIPPED (skips correctly even though it declares no result).
- `anchor-skip` → SKIPPED; `dependent-of-skip` → SKIPPED (cascade still works).
- `anchor-ok`, `dependent-of-ok`, `needs-only-no-dep`, `needed-by-only`,
  `dependent-of-needed-by`, `standalone` → all green.

**Before #982:** `if-only-no-dep` / `needs-only-no-dep` also declared a `skip` result (extra bytes
on every step). **After:** they don't; needed jobs still do; skip still cascades.

---

## 2. cloudbees-automation-controller#552 — `cbp-72007-step-status`

**Build (compile FIRST, then deploy — deploy alone does not rebuild):**
```
curl -s -X POST localhost:10350/api/trigger -H 'Content-Type: application/json' \
  -d '{"manifest_names":["cloudbees-automation-controller-compile"],"build_reason":16}'
# wait for ok, then:
curl -s -X POST localhost:10350/api/trigger -H 'Content-Type: application/json' \
  -d '{"manifest_names":["cloudbees-automation-controller"],"build_reason":16}'
```
Commit the PR branch edits first — a clean tree gives the HEAD-only image tag; uncommitted edits
change the content-hash tag.

**Trigger** `cbp-72007-step-status`, open its run-detail graph.

**Observe — per job:**

| Job | Expected per-step status |
|---|---|
| `overflow-fail` | `s01-fails` → **FAILED**; `s02`..`s20` → **SKIPPED**. None green. |
| `overflow-dependent` | SKIPPED (its need failed). |
| `small-fail` | `only-step-fails` → **FAILED** (short message, no overflow — must NOT become errored). |
| `all-pass` | `p1`,`p2`,`p3` → **SUCCEEDED** (clean messages — guard must not touch them). |

**Before #552:** on `overflow-fail`, the failed step and the skipped steps all show a green check;
only "Finalize job" is red — the real failing step is hidden. **After:** `s01-fails` shows failed,
`s02..s20` show skipped/errored, so the failure is visible on the actual step.

If `overflow-fail` does NOT overflow in this environment (its steps all show correctly even before
#552), add more steps — the overflow is driven by the per-pod termination-message budget being
split across step count. Confirm the raw symptom with:
```
kubectl get taskrun -n "$NS" <overflow-fail-taskrun> -o jsonpath=\
'{range .status.steps[*]}{.name}{"  exit="}{.terminated.exitCode}{"  msglen="}{.terminated.message}{"\n"}{end}'
```
A truncated step shows a non-JSON / cut-off `message` with `exitCode=0`.

---

## 3. scm-providers#105 — `cbp-71978-skip-scm-status`

This only exercises on an SCM-connected `pull_request` run that posts commit statuses — a
`workflow_dispatch` run posts nothing and cannot test it.

**Build.** #105 is in `scm-providers`, consumed by `scm-service`. scm-service uses a `go.work`; to
pick up the local scm-providers PR branch:
1. Ensure `scm-service/go.work` has `use (. ../scm-providers)` (local-only, never commit it), with
   the scm-providers checkout on the PR branch.
2. Reload the Tiltfile, then rebuild scm-service:
```
curl -s -X POST localhost:10350/api/trigger -H 'Content-Type: application/json' \
  -d '{"manifest_names":["(Tiltfile)"],"build_reason":16}'
curl -s -X POST localhost:10350/api/trigger -H 'Content-Type: application/json' \
  -d '{"manifest_names":["scm-service-compile"],"build_reason":16}'
```
> Mirrors the real prod dependency: until scm-service's `go.mod` pins a scm-providers commit with
> #105, scm-service still maps SKIPPED to pending. go.work is the local stand-in for that bump.

**Trigger:** open or push to a PR on the GitHub-connected component, so the `pull_request` event
fires this workflow.

**Observe — on the PR's GitHub checks / commit statuses:**

| Check | Expected |
|---|---|
| `CloudBees/.../skipped-job` | **success** (terminal) — resolves, does not block merge |
| `CloudBees/.../skipped-by-needs` | **success** (terminal) |
| `CloudBees/.../ran-job` | success (control) |

**Before #105:** `skipped-job` / `skipped-by-needs` sit **pending** forever — on a repo with
required status checks the PR is unmergeable. **After:** they resolve to success.

Cross-check from the cluster side that scm-service actually posted a terminal status (not pending):
```
kubectl logs -n <ns> deploy/scm-service | grep -i "build status\|commit status\|segment"
```

---

## Notes / gotchas

- Tilt compiles `CGO_ENABLED=0 GOOS=linux GOARCH=arm64`. Never leave a native `go build -o` binary
  in a deploy path — verify builds via Tilt or with the linux/arm64 flags, not a bare `go build`.
- Run namespaces GC quickly after completion — capture `kubectl get task/taskrun -o yaml` while the
  run is still visible.
- `#982` and `#552` are additive: `#982` alone leaves a depended-on job's failed steps misreported
  (`overflow-fail` is exactly that case — it's needed by `overflow-dependent`); `#552` is what makes
  those steps report correctly. Test them on a build that has both for the full end-to-end.
