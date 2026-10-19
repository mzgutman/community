---
status: proposed
title: Partial PipelineRun Retry
creation-date: '2026-10-13'
last-updated: '2026-10-18'
authors:
- '@mzgutman'
collaborators: []
---

# TEP-0194: Partial PipelineRun Retry

## Summary

This TEP proposes an adaptive **retry** for Tekton Pipelines: reuse eligible
results and re-execute only the failed portion of a completed PipelineRun when
partial retry is possible; otherwise, re-execute the full pipeline when a new
run can be constructed from the source.

A retry is a **new PipelineRun** in the same namespace. The client creates that
object with source-reference metadata. The **controller** computes the failed
subgraph and selects the strategy before scheduling TaskRuns. It uses partial
retry when the subgraph is non-empty and the required source data and workspace
bindings are available to the controller. If partial retry is not possible, it
uses a full retry when it can construct a valid new run from the source. If the
source itself or required run configuration is unavailable, setup fails with
an actionable condition.

**Key capabilities:**
- Compute the failed subgraph (failed tasks + cascaded skips + required dependents)
- Honor the user-requested `--full` or `--partial` strategy; fail closed for `--partial` when required data is unavailable
- Memoize results from bypassed successful tasks (controller-owned status)
- Reuse a durable **user-owned PVC** when the retry still needs workspace files
- Pin the retry to the original resolved Pipeline/Task definitions
- Same planning behavior for `tkn`, Dashboard, and Console (thin clients)

**Example:** `clone → build → test → deploy` fails at `test`. Partial retry:
- Reuses results from `clone` and `build`
- Re-executes `test` and `deploy`
- Does **not** synthesize succeeded TaskRuns for `clone`/`build`

## Motivation

### Goals

**Primary goal:** Reuse successful work through partial retry when the required
plan and data are available; otherwise recover by running the full pipeline
when a valid new run can be constructed.

1. Compute the failed subgraph from skip reasons, result dependencies, and `onError`
2. Make bypassed-task results available to retried tasks without forging TaskRun status
3. Support workspace-dependent retries only when a durable user-owned PVC is still usable
4. Use the same Pipeline/Task definitions as the original run (no Git/OCI drift)
5. Keep planning in the controller so clients stay consistent
6. Record lineage (`retryOf` name + UID) for audit; full Chains integration is later

### Non-Goals

**Out of scope for initial release (alpha):**

1. Recursive partial retry inside nested PipelineRuns (child PRs are atomic)
2. A Kubernetes-routed `/retry` subresource or aggregated API server
3. Automatic retry (`auto-retry` annotations / policies)
4. Cross-namespace retries
5. Automatic adaptive strategy selection without user input; users explicitly choose `--full` or `--partial`
6. User-modified memoized results
7. Full Tekton Chains integration (must not fake skipped-task success; Chains contract is later / Phase 3)
8. Recreating, copying, or re-cloning workspaces; transferring PVC ownership; blocking source cleanup
9. Authoritative server-side dry-run/preview API (clients may offer best-effort preview)
10. Retry after the source PipelineRun itself has been garbage-collected (Phase 2 / Results / artifacts)

### Use Cases

#### Use Case 1: Flaky Test Recovery
Expensive `clone`/`build` succeeded; `test` flakes. Retry reuses build outputs and re-runs tests.

#### Use Case 2: Transient Infrastructure Failure
`apply-manifests` times out after successful build/push/scan. Retry those later stages only, with lineage to the original run.

#### Use Case 3: Credential Expiry at the End
Long pipeline fails on `publish-release` due to token TTL. Refresh credentials and retry the last stages if workspace/results are still available.

**Caveat for all use cases:** Partial reuse requires the source PipelineRun
and required TaskRun/result data to remain available. If TaskRuns or results have
been pruned but the source PipelineRun still contains enough information, the
controller can fall back to a full retry. If the source PipelineRun itself has
been pruned, the live-data design cannot reconstruct the request; environments that prune runs in
minutes (e.g. some Konflux setups) need a later Results/artifact-based design.

### Requirements

#### Functional Requirements

**Retry strategy:**
- The user explicitly selects the strategy via `--full` (default) or `--partial` on the CLI, expressed as a `tekton.dev/retryStrategy: full|partial` annotation on the retry PipelineRun
- **`--full` (default):** Controller always performs a full retry; re-executes all main DAG tasks subject to their `when` expressions. No historical results are required
- **`--partial`:** Controller attempts partial retry. If partial is not possible (non-empty failed subgraph unavailable, required results unreadable, or required PVCs missing/unusable), setup fails with an actionable condition. There is no silent fallback to full retry when the user explicitly requested partial
- Persist the requested and selected strategy before scheduling any TaskRuns; the strategy cannot change after TaskRuns have started

**Failed subgraph:**
- Include explicit failures and interrupted in-flight tasks
- Include skip cascades (`ParentTasksSkip`, `StoppingSkip`, `MissingResultsSkip`, timeout/graceful-stop skips)
- Exclude `WhenExpressionsSkip` (re-evaluate)
- Honor `onError: continue` rules (see Design Details)
- Matrix: if any combination fails, re-run **all** combinations (initial release)
- Exclude finally from the partial subgraph (they run fresh on the new PR); an empty or finally-only partial subgraph selects full retry when possible

**Planning:**
- Controller computes the subgraph, checks whether partial retry has the required data, then selects partial or full retry
- If required historical results cannot be read, do not trust client-supplied values; select full retry when possible
- Clients do not implement subgraph logic
- Do not create synthetic/ghost TaskRuns with forged `Succeeded` conditions

**Result re-injection:**
- Harvest order: (1) `source.status.results[]`, (2) source `TaskRun.status.results[]`, (3) `source.status.retriedTaskResults[]` (inherited memoized results from a prior partial retry, enabling chained retries where earlier TaskRuns may be gone)
- `retriedTaskResults` is populated as the **union** of freshly harvested results and results inherited from `source.status.retriedTaskResults[]`; this carry-forward invariant ensures one-hop lookup is sufficient for arbitrarily deep retry chains
- `retriedTaskResults[].results[]` uses `v1.ResultValue` (the same type as `TaskRun.Status.Results[].Value`), supporting `string`, `array`, and `object` result types
- For bypassed **matrix tasks**, harvest step 2 finds all N source TaskRuns for the pipeline task, sorts them by TaskRun name for deterministic ordering, and aggregates each result name into a single `type: array` entry in `retriedTaskResults`; the array index order is fixed at harvest time and must match the source run's ordering
- Persist accepted memoized results in **status** (not as the user-writable source of truth)
- Partial retry requires all required results; if any are missing or unreadable, setup fails with an actionable condition for `--partial`
- Respect the ~4KB result size limit; large payloads belong in artifacts/CAS, not result fields
- - **Authz:** Harvest only what the controller is already allowed to read.
  Tekton Results is out of alpha/beta. If required results are missing/unreadable, full retry
  when possible; never trust unsigned client-supplied values or forge TaskRun
  success. Phase 2 may fetch payloads via a caller-authorized verified snapshot;
  planning remains server-side.

**Workspace (alpha/beta policy):**
- Supported path: **opt-in reuse of an existing user-owned PVC** explicitly listed
  by the user on the retry PipelineRun, in the retry's namespace, whose lifetime
  does not depend on the source run
- The source PipelineRun must be terminal. Tekton does not snapshot or verify
  the filesystem contents and makes no reproducibility claim for PVC reuse.
- Do not recreate, copy, re-clone, or reassign ownership of the workspace.
- If a workspace needed for partial reuse is missing or unusable, select full
  retry when the new run can execute with valid workspace bindings; otherwise
  fail with an actionable condition.
- If the claim is ReadWriteOnce and busy, keep the retry `Pending` and requeue
  until a configured deadline. If it remains busy, fail with an actionable
  deadline condition.
- Do not create retry TaskRuns while another visible active Tekton run uses the
  claim. This is Tekton-level best-effort exclusivity; storage enforcement and
  Pods outside Tekton are out of scope.

**User-owned PVC selection (alpha/beta):**
- User must explicitly list `spec.workspaces[].persistentVolumeClaim` on the retry PipelineRun
- The PVC must exist in the same namespace (validated by admission)
- The PVC name is typically copied from the source or environment config
- CLI/UI may suggest available PVCs in the namespace (best-effort)

**Definition stability:**
- Execute `source.status.pipelineSpec` (resolved definition), not a live mutable `pipelineRef`
- Provenance annotations (Git SHA / OCI digest) for audit

**API shape:**
- Retry is requested by creating a new PipelineRun with paired source references
- No `/retry` HTTP API in alpha or beta

#### Non-Functional Requirements

**Performance:** Planning should run in-cluster (informers/caches), not as N client-side graph walks.

**Security and admission requirements:**
- Same namespace for source PipelineRun, retry PipelineRun, and referenced PVCs
- Caller must be allowed to `get` the source PipelineRun and `create` the retry
- Source UID must match `retryOfUID`, and the source must be terminal
- Retry lineage metadata and source references are immutable after creation
- Clients cannot write controller-owned retry status or memoized results
- The controller must not forge TaskRun status for bypassed tasks
- Alpha/beta harvest uses only the controller's existing authorization; unreadable
  historical data is never trusted and selects full retry when possible
- **RBAC enforcement for harvest:**
  - Controller uses SubjectAccessReview (SAR) to check `get` TaskRuns in the namespace
  - If controller cannot read a required TaskRun.status.results[], partial retry is unavailable
  - No fallback to client-supplied values (fail-closed)
- Admission/SAR design must enforce these rules before production

**Exclusivity layers:** Tekton prevents assignment when a visible active Tekton
run uses the PVC; this is Tekton-level best effort. Storage-backend enforcement
and filesystem consistency are separate concerns, and Pods or workloads outside
Tekton are out of scope.

**"Visible active Tekton run" definition:**
- PipelineRun or TaskRun in the same namespace whose Pod(s) are still bound to the PVC
- Determined by checking PVC usage in Pod.status (best-effort; not guaranteed by storage driver)

**Auditability:**
- Lineage annotations on the retry object
- Kubernetes Events
- Distinct retry vs normal-run observability via metadata/status

**Backward compatibility:**
- Normal PipelineRuns unchanged
- Feature gated behind `enable-api-fields` flag, graduating alpha → beta → stable

## Proposal

**Terminology used in this TEP:**
- **alpha / beta / stable** — feature graduation via `enable-api-fields` (Implementation Plan Phase 1 / 1b).
- **live-data path** — the initial design that harvests from live cluster objects while the source PipelineRun and related data still exist.
- **Phase 2** — later Results / artifact-backed persistence that can survive source garbage collection.
- `apiVersion: tekton.dev/v1` — the Kubernetes API version; unrelated to the terms above.

Introduce partial retry by:

1. **Representing a retry as a new PipelineRun** the client creates, for example:

```yaml
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  name: build-42-retry-1
  annotations:
    tekton.dev/retryOf: build-42
    tekton.dev/retryOfUID: 6fc8e6e6-1cc3-4fa5-bb3e-7c1b2e0b637a
    tekton.dev/retryStrategy: partial   # "full" (default) or "partial"; set by CLI flag --full/--partial
spec:
  params: []       # must be identical to the source run for partial retry; any diff causes setup failure
  workspaces:
    - name: shared
      persistentVolumeClaim:
        claimName: build-workspace  # explicitly selected by the user
```

   Recommended name: `<source-name>-retry-<n>`.

2. **Controller-side setup before scheduling TaskRuns:**
   - Resolve source by name and verify UID
   - Snapshot `source.status.pipelineSpec` onto the retry (controller-owned)
   - Read the requested strategy from `tekton.dev/retryStrategy` annotation (`full` if absent)
   - For `partial`: compute the failed subgraph, verify required results and workspace bindings, verify effective params match the source; if any check fails, setup fails with an actionable condition (no fallback to full)
   - For `full`: mark every DAG task for execution; no historical results or param matching required
   - Persist `status.retryMetadata.strategy` and reason before creating any TaskRuns; strategy is immutable thereafter
   - For partial retry, harvest results into **status** memoization fields (union of fresh harvest and `source.status.retriedTaskResults[]`) and validate required user-owned PVCs
   - Then create TaskRuns for the selected task set

3. **Extending result resolution** so bypassed tasks resolve from memoized
   **status** (and live TaskRuns of this retry), never from forged child TaskRuns.

4. **Thin clients:**
   - `tkn pipelinerun retry <name>` defaults to `--full`; pass `--partial` to request partial retry (fail-closed if conditions are not met)
   - Dashboard / Console replace the existing **Rerun** button with a single **Retry** action that includes a full/partial selector (dropdown or toggle, defaulting to full); after create, link to the new PipelineRun

   **Current full Rerun baseline** (see architecture-analysis Part 5): today Dashboard **Rerun**, Console **Rerun** / **Start last run**, and `tkn pipeline start --last` / `--use-pipelinerun` (plus optional `tkn pipelinerun export`) all clone client-side with **no** Pipelines Rerun API and **no** `retryOf` lineage. There is no `tkn pipelinerun retry` yet. The new Retry action **replaces** the existing Rerun button in Dashboard and Console; `tkn pipeline start --last` remains available in the CLI for scripts that do not need lineage but has no UI equivalent.

### Notes and Caveats

**Workspace reuse:** Reuse is opt-in and refers to the current contents of the
user-owned PVC after the source run is terminal. Tekton does not snapshot,
rollback, or verify filesystem consistency, and makes no reproducibility claim.
This is a narrow live-data compatibility path, not the general persistence model.

**`onError: continue` result staleness:** If a failed continue-task emitted a result that a successful downstream consumed, that downstream is not in the subgraph; a different result on retry can leave that work stale.

**Side effects:** Tekton does not detect non-idempotent tasks. Docs/UX warn; the feature does not block automatically.

**User-owned PVC only:** `volumeClaimTemplate` claims owned/cleaned up with the source run are **not** guaranteed in alpha/beta. If partial retry needs a missing claim, select full retry only when the new run can be constructed with valid workspace bindings; do not reconstruct or transfer the source-owned claim.

**RWO busy behavior:** A retry waits in `Pending` and requeues while a visible
Tekton run still uses the claim. After the configured deadline, it fails with
an actionable condition. A Lease, dedicated coordination resource, or stronger
cross-controller locking protocol is deferred unless implementation experience
shows that best-effort exclusivity is insufficient for the initial release.

**Clarification: Source cleanup and RWO wait:**
- Retry **detects** when source is still using the claim (observed state)
- Retry **waits** but does not prevent/block source cleanup operations
- If source completes and releases the claim before deadline, retry can proceed
- If deadline expires while source is gone, retry fails (not a success)

**Pruning:** If source TaskRuns or required results are gone while the source
PipelineRun and resolved definition remain, partial retry may be unavailable;
the controller selects full retry when possible. If the source PipelineRun or
required run configuration is gone, the live-data design cannot reconstruct the retry. Durable
post-GC retry is a later Results/artifact design.

**Task-level retry budget:** Each retry PipelineRun is a new execution; `spec.retries` budgets reset.

**No status forgery:** Bypassed tasks must not appear as succeeded TaskRuns of the retry. Memoization is explicit retry status, not fake child runs (required for Chains-safe design).

## Design Details

### Failed Subgraph Computation

The controller applies these rules to the source `PipelineRunFacts`:

**Rule 1 - Explicit failures:** `Succeeded=False`, or interrupted in-flight when the PR was cancelled/timed out.

**Rule 2 - Skip cascades:** `ParentTasksSkip`, `StoppingSkip`, `MissingResultsSkip`, `PipelineRunTimeoutSkip`, `GracefullyStoppedSkip`.

**Rule 3 - Intentional skips:** Exclude `WhenExpressionsSkip` (re-evaluate).

**Rule 4 - `onError: continue`:**
- Failed task: included
- Downstream succeeded using that result: excluded (staleness risk)
- Downstream validation-failed (missing result): included
- Downstream `runAfter` only and succeeded: excluded

**Rule 5 - Matrix (initial release):** Any combination failed → re-run all combinations.
- Rationale: Matrix expansion happens at task level; no partial-within-matrix in alpha/beta.
- Future: a later release could support per-combination retry if needed.
- **Bypassed matrix tasks:** If all combinations of a matrix task succeeded and it is preserved (not in the subgraph), the controller must still make its aggregated results available to any re-running downstream tasks that reference `$(tasks.X.results.Y[*])` or `$(tasks.X.results.Y[i])`. See the Result Re-injection section for the required harvest and state-machine handling.

**Rule 6 - Finally:** Not in the partial subgraph; they run fresh on either retry strategy. If *only* finally failed, the partial subgraph is empty and the controller selects a full retry when possible.

**Rule 7 - Custom tasks:** Atomic (no recursion into the custom controller).

**Rule 8 - Nested PipelineRuns:** Atomic (no inner partial retry).

**Rule 9 - Task-level retries:** Fresh budget on the new PipelineRun.

**Example:**

```
Original:
  A (build ✓) → B (test ✗, onError: continue) → D (deploy ⊘ ParentTasksSkip)
  A → C (lint ✓) → E (report ✓, uses B.result)

Failed subgraph: {B, D}
Reused (memoized, not re-run): {A, C, E}
```

**Example status (partial retry):**

```yaml
status:
  retryMetadata:
    strategy: partial
    preservedTasks: ["A", "C", "E"]
    rerunTasks: ["B", "D"]
  retriedTaskResults:
    - pipelineTaskName: A
      results:
        - name: build-sha
          value: "abc123def456"
    - pipelineTaskName: C
      results:
        - name: lint-status
          value: "passed"
```

### Result Re-injection (Memoization)

**Problem:** The retry PR has no child TaskRuns for bypassed tasks, so
`$(tasks.X.results.Y)` has no live TaskRun.

**Where results live vs who can read them:** Harvest is not only a data-structure
problem. Access depends on **storage** and **identity**. In the live-data path (alpha/beta), only the
controller's existing authorization is used:

| Store | Typical reader | Controller harvest? |
| --- | --- | --- |
| Source `PipelineRun.status.results[]` | Anyone who can `get` that PR | Often yes, if the controller already reconciles that namespace |
| Source `TaskRun.status.results[]` | `get`/`list` TaskRuns in that namespace | **Not guaranteed** (namespace-scoped installs, extra RBAC, objects already GC’d) |
| Source `PipelineRun.status.retriedTaskResults[]` | Anyone who can `get` that PR | Yes — controller-owned status, same authorization as reading the source PR; enables chained retries where earlier TaskRuns are gone |
| Tekton Results records | Results API authz (often per-tenant / user) | **Not an alpha/beta store.** Phase 2 access is caller-authorized; the Pipelines controller SA is not assumed to query all namespaces’ stored runs. |
| Artifacts / CAS | Store-specific credentials | Same: do not assume cluster-wide controller access |

**Implication:** The controller should harvest only what its **own** RBAC allows.
If required historical data is unreadable, partial retry is unavailable; the
controller selects full retry when it can construct one. A client-supplied
payload is never a fallback and must not impersonate TaskRun success.
Caller-authorized fetching is reserved for the Phase 2 snapshot protocol below.

Split two jobs that the first drafts lumped together:

1. **Plan (subgraph)** — keep **server-side** for alpha/beta (one implementation, no client skew).
2. **Fetch result bytes** — in the live-data path, the controller fetches only when authorized.
   If required bytes are unavailable, it selects full retry when possible. In
   Phase 2, a caller-authorized client or broker may fetch historical data and
   provide a verified snapshot for partial retry.

**Live-data default (cluster objects the controller can already see):** Controller copies
into **status** memoization (e.g. `status.retriedTaskResults`).

**Harvest order (during setup):**
1. **Bubbled results** from `source.status.results[]` (preferred)
2. Source `TaskRun.status.results[]` (if available and readable)
3. `source.status.retriedTaskResults[]` — inherited memoized results from a prior partial retry (enables chained retries; earlier TaskRuns may be gone)

**Matrix task memoization (harvest step 2 extension):**
For a bypassed matrix pipeline task (all combinations succeeded, task is in `preservedTasks`):
- Find **all N source TaskRuns** for that pipeline task using the `tekton.dev/pipelineTask` label.
- Sort them by TaskRun name (lexicographic ascending), matching the ordering produced by the existing `createResultsCacheMatrixedTaskRuns` function in the reconciler.
- For each result name, aggregate the scalar string values from all N TaskRuns into a single ordered array.
- Store as a `type: array` result entry in `retriedTaskResults` (using `v1.ResultValue{Type: ParamTypeArray, ArrayVal: [...]}`).

The array index ordering is fixed at harvest time. Array index 0 in the memoized result must correspond to the same matrix combination as index 0 in the source run. This invariant is required for downstream tasks that reference individual indices such as `$(tasks.build-images.results.IMAGE_REF[0])`.

**Carry-forward invariant:** `retriedTaskResults` on each retry is populated as the **union** of freshly harvested results and results inherited from `source.status.retriedTaskResults[]`. This ensures one-hop lookup is sufficient for arbitrarily deep retry chains: `retryOf` always points to the immediate parent, and that parent's `retriedTaskResults` already contains the complete set needed by its preserved tasks.

**Resolution order (during execution of `ResolveResultRef()`):**
1. Live TaskRuns of **this** retry
2. Memoized retry status for bypassed tasks

**`isBypassed()` predicate and state machine impact:**
Tekton's existing result-resolution guards — in `convertToResultRefs`, `GetTaskRunsResults`, and related functions — gate on `isSuccessful() || isFailure()`. A bypassed task has no live child TaskRuns on the retry PipelineRun, so both predicates return false. Without additional handling, these guards produce an error ("task was not finished") even when the task's results are fully memoized in `retriedTaskResults`.

The implementation must introduce an `isBypassed()` predicate on `ResolvedPipelineTask`, set to `true` for every task in `preservedTasks` during retry setup. Every guard that currently checks `isSuccessful() || isFailure()` in a result-resolution path must be extended to also pass when `isBypassed()` is true, resolving from memoized status instead of live TaskRun objects.

For bypassed **matrix tasks** specifically, the controller must pre-populate `ResolvedPipelineTask.ResultsCache` from the aggregated array stored in `retriedTaskResults` before any TaskRun scheduling begins. The existing `findResultValuesForMatrix` → `ResultsCache` lookup then works correctly once the cache is seeded and the `isBypassed()` gate is respected.

**Semantic change to `GetTaskRunsResults()`:** Once bypassed tasks satisfy the `isBypassed()` gate, `GetTaskRunsResults()` will include them alongside tasks with live TaskRuns. The callers of this function are:
- `ApplyTaskResultsToPipelineResults` — pipeline-level result computation; correctly resolves pipeline results that reference bypassed tasks' memoized values.
- `ApplyResultsToWorkspaceBindings` — workspace binding substitution; array results are already skipped (`case v1.ResultsTypeArray: continue`), so no side effect.
- `PropagateResults` — implicit result propagation to downstream tasks; correctly propagates memoized array results.

This behavior is correct and desired, but must be documented internally: entries in `GetTaskRunsResults()` for bypassed tasks are sourced from memoized status and have no backing live TaskRun object. Code that iterates this map must not unconditionally dereference a live TaskRun for every entry.

`isBypassed()` is only ever set to `true` during retry setup for tasks in `preservedTasks`. Normal (non-retry) PipelineRuns never set this predicate, so existing behavior for all non-retry runs is unchanged.

**Execution-status context variables for bypassed tasks (`$(tasks.X.status)`):**
Finally tasks may guard their `when` expressions on `$(tasks.X.status)` (e.g., `$(tasks.build.status) == "Succeeded"`) or the aggregate `$(tasks.status)`. When task X is bypassed in a partial retry, no live TaskRun exists for X on the retry PipelineRun. The execution-status context map must therefore be seeded with `"Succeeded"` for each bypassed task before finally tasks are scheduled. The aggregate `$(tasks.status)` is computed from the union of live and bypassed task statuses. This is an extension of the `isBypassed()` predicate: the controller must populate the status context map for bypassed tasks in addition to the result-resolution path.

If the controller **cannot** read required objects, it must not silently skip
memoization or trust client-provided values. For `--partial` requests this is
a hard failure (fail closed); there is no silent fallback to full retry when
the user explicitly requested partial. A later Phase 2 contract may allow a
caller-authorized service to attach a verified snapshot with enough provenance
for the controller to accept the values without forging TaskRun conditions.

**Phase 2 direction:** Let the Results or artifact service provide a verified snapshot of the source run’s results, authorized for the caller and bound to the source PipelineRun and retry plan. The controller verifies the snapshot before using it. If the caller cannot obtain one, partial retry is unavailable; 
the controller uses full retry when possible.
The snapshot can carry small values inline and reference large artifacts by content digest and opaque store reference. The retry API stays storage-neutral; OCI-backed storage is one possible implementation (TEP-0193).

**Limitations:** Result fields are limited to about 4 KB; larger values must use artifacts.

### Retry Orchestration Boundary

**Decision:** Server-side **subgraph planning**. Clients create the retry
PipelineRun. 

**Planning vs harvest (authz):**

Planning and fetching historical results are separate (least privilege). The
Pipelines controller is not granted cluster-wide Results access for all tenants.

- **Live-data (alpha/beta):** Controller harvests only what its existing RBAC allows. Clients do not
  reimplement subgraph rules. Unreadable required results → full retry when
  possible; never accept unsigned client-supplied values or forge TaskRun success.
- **Phase 2 handoff:** Controller publishes the authoritative plan and required
  results. A client or broker fetches with the **caller's** credentials and
  returns an immutable snapshot. The controller verifies binding to source UID,
  plan, namespace/tenant, and retention/expiry, then memoizes accepted values in
  retry **status**. It does not impersonate the caller or retain their token.
- **Chains-safe:** The snapshot attests authorized observation for reuse — not
  that a retry TaskRun succeeded.
  - Memoized results in retry status must be clearly marked as inherited/reused (not fresh execution)
  - Chains must not attest memoized results as if the retry TaskRun produced them
  - Chains policy: Attest only TaskRuns created by this retry, not bypassed source tasks
- **Large data:** Inline small results; large artifacts as opaque content
  digests. Storage URI/retrieval stay with the authorized integration so the
  retry API stays backend-agnostic (e.g. OCI).
  
### API Shape

**Decision:** The retry **is** the new PipelineRun. No `POST .../retry` API in alpha or beta.
Clients request a strategy via the `tekton.dev/retryStrategy` annotation (`full` or `partial`);
the controller records the selected strategy and reason in status before scheduling children.

Dry-run/preview needs a separate design. Clients may estimate a plan locally
for UX but that is **not** authoritative.

A dedicated `PipelineRunRetry` CRD remains a possible later option if request
and execution must be split.

### Workspace and PVC Handling

**Alpha/beta policy:** Reuse existing **user-owned PVCs** only. No copy, re-clone,
ownership transfer, or blocking source cleanup.

If partial retry needs workspace files and the claim is gone, select full retry
when the pipeline can run with valid workspace bindings. Otherwise fail with
guidance (use a durable user-owned PVC, or create a new run with valid
workspace configuration).

`volumeClaimTemplate` / source-owned claims that GC with the source run are
not guaranteed in alpha/beta.

**RWO:** Do not assume two PipelineRuns can share a ReadWriteOnce claim.
The controller must not assign the claim to a retry while another visible
active Tekton run uses it. The retry remains `Pending` and waits with backoff
until the configured deadline, then fails with an actionable condition.
This is Tekton-level best-effort exclusivity only; the storage backend and
non-Tekton Pods are outside the guarantee.

**Deadline configuration:**
- Configurable via controller flag (default: 5 minutes or TBD per implementation)
- Stored in retry status for observability
- Can be overridden per-retry via annotation (future enhancement if needed)

### Pipeline Definition Stability

Copy/set `retry.spec.pipelineSpec` from `source.status.pipelineSpec` before
scheduling (controller-owned snapshot). Provenance annotations record Git SHA /
OCI digest for audit. Exact mutation/compare rules are a follow-up API contract
(semantic compare after defaulting; users must not swap the definition after
create).

### Data Availability

#### Live cluster data (alpha/beta)

| Retry type | Required data | If unavailable |
|---|---|---|
| Any retry | Source PipelineRun and resolved pipeline definition | Fail setup |
| Partial retry | Eligible failed subgraph, required readable results, and any needed PVC explicitly supplied on the retry | Use full retry if possible; otherwise fail setup |

PVC reuse is opt-in. The user must provide a same-namespace, user-owned PVC in the retry spec. Tekton uses its current contents; it does not snapshot or guarantee reproducibility. Source-owned claims are not transferred.
The source PipelineRun, TaskRuns, results, and PVC may be removed at different times. Partial retry is available only while all required data remains present and readable.

#### Phase 2: Data that survives source cleanup

Phase 2 may use Results or an artifact service to preserve and provide the data needed for partial retry after the source run is removed. The service should authorize access for the caller, and the controller should verify the data before using it. If verified data is unavailable, the controller uses full retry when possible.

The design should work with different storage backends; OCI is one possible option. The Pipelines controller should not need broad Results access or direct registry access. PVC reuse remains a live-data mechanism, not durable Phase 2 storage.

### API Design

**Lineage and strategy request (client-set, immutable after create):**

```yaml
metadata:
  annotations:
    tekton.dev/retryOf: build-42
    tekton.dev/retryOfUID: 6fc8e6e6-1cc3-4fa5-bb3e-7c1b2e0b637a
    tekton.dev/retryStrategy: partial   # "full" (default if absent) or "partial"
```

`retryOf` is the **immediate parent name**; `retryOfUID` is that object's UID.
`tekton.dev/retryStrategy` is set by the CLI (`--full`/`--partial`) and is immutable after create.
`tekton.dev/retryAttempt` is **controller-derived**, not a client-chosen
arbitrary number.

**Status (controller-owned):**

```yaml
status:
  retryMetadata:
    retryOf: build-42
    retryOfUID: 6fc8e6e6-1cc3-4fa5-bb3e-7c1b2e0b637a
    retryAttempt: 1
    strategy: partial # or full
    strategyReason: required-data-available
    preservedTasks: ["clone", "build"]
    rerunTasks: ["test", "scan", "deploy"]
  retriedTaskResults:
    - pipelineTaskName: clone          # scalar string result
      results:
        - name: commit-sha
          value: "abc123def456"
    - pipelineTaskName: build-images   # bypassed matrix task: aggregated array result
      results:
        - name: IMAGE_REF
          type: array
          value:
            - "quay.io/org/img@sha256:aaa"   # index 0 — first combination by TaskRun name sort
            - "quay.io/org/img@sha256:bbb"
            - "quay.io/org/img@sha256:ccc"
            - "quay.io/org/img@sha256:ddd"
```

Field names are indicative; CRD types will be finalized in implementation.

`retriedTaskResults[].results[]` uses `v1.ResultValue` (the same type as `TaskRun.Status.Results[].Value`), supporting `string`, `array`, and `object` result types. For a bypassed matrix task, the N scalar results produced by N TaskRun combinations are aggregated into a single `type: array` entry; the array index ordering matches the source run's TaskRun name sort order and is immutable after harvest.

For `strategy: partial`, `preservedTasks` identifies tasks whose results are
reused and `rerunTasks` identifies tasks selected for fresh execution. For
`strategy: full`, all main DAG tasks are evaluated afresh (subject to their
`when` expressions), `preservedTasks` is empty, and `strategyReason` explains
why partial retry was unavailable.

**Scheduling rule:** No retry TaskRuns until setup has persisted the definition
snapshot and selected strategy. For partial retry, required memoized results
must also be ready. The strategy cannot change after TaskRuns are scheduled.

**Strategy selection and immutability:**
- Strategy is selected and persisted in status during the **first reconciliation** of a retry PipelineRun
- Before any TaskRuns are created
- Immutable thereafter (enforced via admission webhook or field validation)

**Admission / security:** The mandatory checklist is:

*Validating webhook rules:*
- Source UID matches `retryOfUID`
- Same namespace for retry, source, and PVCs
- Lineage annotations (`retryOf`, `retryOfUID`, `retryStrategy`) immutable after create (enforced by admission webhook)
- `spec.pipelineSpec` immutable after creation (enforced by admission webhook)
- No cross-namespace PVC references
- `tekton.dev/retryStrategy` must be `full` or `partial` if set; defaults to `full` when absent

*Runtime SAR checks (during reconciliation):*
- Controller can `get` source PipelineRun
- Controller can `get` source TaskRuns (if needed for harvest)
- Caller could `get` source and `create` retry (verified at creation time)
- For `--partial`: effective params on retry must match effective params of source (resolved against `source.status.pipelineSpec` defaults); mismatch causes setup failure with `strategyReason: parameter-mismatch`

*General:*
- Clients cannot write controller-owned retry status or memoized results
- Controller-owned status only
- For `--partial`: unreadable required results cause setup failure (fail closed); there is no silent fallback to full retry
- Never accepted from an unsigned client payload

**RBAC (illustrative):**
- **Caller:** `get` source PR, `create` PipelineRuns.
- **Controller SA:** existing reconcile rights only (not cluster-wide Results).

### CI Integration (e.g. Pipelines-as-Code)

Integrations can identify a retry by inspecting the `tekton.dev/retryOf` and
`tekton.dev/retryOfUID` annotations, which are client-set and immutable after
creation.

**Check / commit status behavior:**
- Checks are scoped to **commits**, not PipelineRuns. A retry should be
  associated with the original commit's check, not with any newer commit, even
  if the PR has advanced since the original execution.
- A new execution triggered by a new push should create a check for that new commit.
- For a retry, the integration may either update the original commit's existing
  check run or create a separate attempt entry on the same commit. Each is a
  valid policy: updating gives a cleaner single-result view; a separate entry
  preserves the full attempt history.

**New commits since the original run:**
- If the PR has advanced since the source run, the retry still refers to the
  original commit. Whether to warn, block, or proceed is left to the integration.
- Partial reuse of results from a different commit is semantically unsound;
  integrations that detect this condition should surface it clearly.
- Detecting a PR advance requires knowing which commit the source run tested.
  If the TEP's provenance annotations record that commit SHA, integrations can
  compare it against the PR's current HEAD; otherwise the integration must
  obtain that information through its own means (e.g. the triggering event).

**What the TEP provides to integrations:**
- `tekton.dev/retryOf` / `tekton.dev/retryOfUID` — identify retry lineage
- `status.retryMetadata.strategy` — `full` or `partial`
- `status.retryMetadata.preservedTasks` / `rerunTasks` — which tasks ran fresh

The exact check-status policy is integration-specific and out of scope for the initial release.

## Design Evaluation

### Reusability

Builds on PipelineRun, `status.pipelineSpec`, `PipelineRunFacts` / DAG, and
`ResolveResultRef()`. Retry is runtime-only; pipelines need no authoring change
except optional result bubbling and durable PVCs if workspace reuse is required.

### Simplicity

**Without (today):** client-side full clone — Dashboard/Console **Rerun**, or
`tkn pipeline start --last` / `--use-pipelinerun` / `pr export` + create — always
re-executes the full DAG with no controller strategy or lineage
(architecture-analysis Part 5).

**With:** `tkn pipelinerun retry <name>` (default `--full`) or `tkn pipelinerun retry <name> --partial` for explicit partial reuse. The controller executes the requested strategy; `--partial` is fail-closed if conditions are not met.
Dashboard and OpenShift Console **replace** the existing Rerun button with a single **Retry** action that includes a full/partial selector, defaulting to full. The `tkn pipeline start --last` path remains in the CLI only, for scripts that do not need lineage.

Minimal alpha/beta surface: annotations + status plan/results + reconciler setup path.
No aggregated API, no new CRD required for alpha/beta.

### Flexibility

No Results/Chains/artifact store is required for alpha/beta. A later release may use caller-authorized
Results and trusted artifacts/snapshots. Opinionated: server-side plan; matrix
all-or-nothing; user-owned PVC only.

### Conformance

Uses ordinary PipelineRun create/watch/status. Does **not** claim a `/retry`
subresource like `/status` or `/scale`.

### User Experience

One Retry action on a terminal unsuccessful PipelineRun. The user explicitly selects the strategy: `--full` (default) for a clean full re-run, or `--partial` to reuse eligible preserved results. With `--partial`, if the controller cannot plan a valid partial retry (missing results, param mismatch, unusable PVC), setup fails with an actionable error — there is no silent fallback to full retry.

Clients create the retry PipelineRun with lineage, source params, the `tekton.dev/retryStrategy` annotation, and any same-namespace user-owned PVC binding. Status reports `strategy` / `strategyReason`, preserved vs rerun tasks, and setup errors (including RWO wait/deadline).
Docs warn that PVC reuse is as-is (not a snapshot), that `--partial` requires identical params to the source, and that non-idempotent tasks may have side effects on re-run.

**How users understand retry strategy:**
- Status shows `strategy: partial` or `strategy: full` + `strategyReason`
- Example reasons: `required-data-available`, `missing-results`, `parameter-mismatch`, `rwo-busy-deadline`, `workspace-unavailable`
- Users can inspect `status.preservedTasks` and `status.rerunTasks` to understand what was reused

**Dashboard / OpenShift Console:** Replace the existing **Rerun** button with a single **Retry** action on terminal unsuccessful PipelineRuns. The action includes a full/partial selector (dropdown or toggle) defaulting to **full**. Selecting **partial** and clicking Retry creates a PipelineRun with `retryOf` lineage and `tekton.dev/retryStrategy: partial`; the UI surfaces the setup failure if partial conditions are not met. After create, link to the new PipelineRun.

**CLI:** `tkn pipelinerun retry <name>` defaults to `--full`. Use `--partial` to request partial retry; the command fails if the controller cannot satisfy the partial request. `tkn pipeline start --last / --use-pipelinerun` remains available for scripts that do not need lineage — it has no `retryOf` annotation and is unaffected by this design.

### Performance

Setup uses in-cluster reads (same order of magnitude as one reconcile), not
three independent client graph implementations. Exact latency targets will be
validated in tests; the architectural win is **one** planner.

### Risks and Mitigations

| Risk | Mitigation |
|------|------------|
| Side effects | Warn; document idempotent tasks; no auto-retry |
| Workspace consistency | Warn; opt-in PVC reuse makes no snapshot or reproducibility guarantee |
| Partial-reuse data or PVC unavailable | Fail closed with an actionable condition; no silent fallback to full when `--partial` was requested |
| Parameter changes invalidate preserved results | `--partial` requires effective params to be identical to the source; any diff causes setup failure with `strategyReason: parameter-mismatch` |
| RWO busy | Pending + requeue until deadline if the selected strategy uses the claim |
| Tekton/PVC exclusivity | Prevent assignment when a visible Tekton run uses the claim; storage and other Pods are out of scope |
| Source pruned quickly | Document live-data window; Phase 2 Results/artifacts |
| Result size | 4KB limit; artifacts for large data |
| Chains false attestation | No forged TaskRun success |
| `onError: continue` staleness | Documented initial-release tradeoff |
| Unauthorized historical results | Do not reuse them; fail closed for `--partial`. Phase 2 may enable partial retry through a verified, source- and plan-bound snapshot |
| Chained retry after earlier TaskRuns are GC'd | Carry-forward invariant: each retry's `retriedTaskResults` is the union of fresh harvest and `source.status.retriedTaskResults[]`; one-hop lookup is always sufficient |
| Bypassed matrix task results unavailable to downstream re-running tasks | Harvest step 2 aggregates all N source TaskRuns for the matrix task into an ordered `type: array` entry in `retriedTaskResults`; `isBypassed()` predicate added to `ResolvedPipelineTask` and threaded through all `isSuccessful() \|\| isFailure()` guards in result-resolution paths; `ResultsCache` pre-populated from memoized array before TaskRun scheduling; `GetTaskRunsResults()` semantics updated to include bypassed-task entries (sourced from memoized status, not live TaskRuns); no effect on non-retry PipelineRuns |
| `$(tasks.X.status)` / `$(tasks.status)` incorrect for bypassed tasks in finally `when` | Controller seeds the execution-status context map with `"Succeeded"` for each bypassed task before scheduling finally tasks; aggregate `$(tasks.status)` is computed from the union of live and bypassed statuses |

### Drawbacks

1. Controller/reconciler must understand retry setup (not a zero-touch normal PR).
2. Inline `pipelineSpec` increases object size (usually acceptable vs etcd limits).
3. Async UX (create then watch) vs a single HTTP result.
4. Alpha/beta workspace and TTL limits will not cover all Konflux-style prune-heavy clusters.
5. Users can still full-rerun; partial retry is an efficiency feature with constraints.

## Alternatives

### Alternative 1: Client-side planning
Each client computes the subgraph and applies YAML. **Rejected for the initial release:**
duplication, version skew, weaker consistency (ADR: Planning Boundary).

### Alternative 2: HTTP `/retry` subresource
**Rejected for the initial release:** Kubernetes-routed custom API needs an extra API server
(aggregation). (ADR: API Shape.)

### Alternative 3: Synthetic TaskRuns
Ghost TaskRuns with pre-filled success/results. **Rejected:** forges status,
breaks Chains/Pod assumptions.

### Alternative 4: Auto-retry annotations
**Deferred:** needs policies and loop protection.

### Alternative 5: Recreate/copy workspace or steal PVC ownership
**Deferred:** high complexity; couples to cleanup. Alpha/beta = user-owned PVC only.

### Alternative 6: Grant the Pipelines controller broad Results access
**Rejected as the general Phase 2 solution:** it expands tenant permissions, couples
the controller to deployment-specific Results authorization, and may still not
represent the end user's permission to reuse a result. A caller-authorized,
verified result snapshot preserves least privilege while retaining one
server-side planner.

## Implementation Plan

### Phase 1 (Alpha)

Gated behind `enable-api-fields: alpha`.

1. Failed subgraph library + tests (rules 1–9)
2. CRD/status fields for retry metadata and memoized results; lineage annotations
3. Reconciler setup path (source verify, definition snapshot, strategy selection, required harvest/workspace checks, then schedule partial or full task set)
4. Admission/SAR enforcement for source refs, same-namespace scope, terminal
   source, immutable lineage, controller-owned status, and PVC scope
5. Definition snapshot from `status.pipelineSpec`
6. `tkn pipelinerun retry`, Dashboard/Console create+watch
7. Docs: TTL, opt-in PVC reuse, Pending/deadline behavior, no reproducibility
   claim, idempotency warnings, and no status forgery

Deliverable: K8s-only retry while source data still exists, available under the alpha flag.

### Phase 1b (Beta)

Promote to `enable-api-fields: beta` once the alpha implementation is stable and feedback from early adopters is incorporated. Beta graduation requires:
- No breaking API changes since alpha
- Test coverage for all subgraph rules, harvest order, chained retries, and fail-closed partial behavior
- CLI (`tkn pipelinerun retry`) and Dashboard/Console Retry action shipping and tested

### Phase 2

Authorization-aware Results and/or trusted-artifact integration. Persist retry
metadata before source garbage collection where an opt-in retry record or
Results/artifact retention policy is used. The controller
continues to compute the plan, while a caller-authorized Results service or
namespace-scoped broker produces a verifiable, source- and plan-bound result
snapshot. The controller consumes only verified snapshots or data it can read
under its existing permissions; it does not receive cluster-wide Results
access. Retention, expiry, revocation, large-value references, and the exact
snapshot transport are defined by the Phase 2 API contract. Optional result-bubbling
documentation may extend the live-data availability window but does not replace the
authorization protocol.

### Phase 2 (Optional) — Integration with OCI artifacts (TEP-0193)

When TEP-0193 (Tekton Artifacts OCI Storage) is stable:
- Allow retry to harvest from artifact metadata (not just live TaskRuns)
- Persist retry metadata before source garbage collection
- Enable retry across cluster-GC boundaries via OCI-backed snapshots
- Caller-authorized snapshot protocol (least privilege)

**Note:** Alpha/beta does NOT require TEP-0193:
- Retry logic is agnostic to how results are stored (status fields, TaskRuns, future artifact stores)
- Alpha/beta uses existing cluster RBAC and data sources; no new infrastructure needed
- This integration is a future enhancement, not a blocker for alpha/beta adoption

### Phase 3

Chains contract for inherited vs fresh work; matrix/policy/auto-retry extras.

### Test Plan

Unit: subgraph rules; strategy selection for eligible partial retry,
unavailable results or workspace, empty/finally-only subgraph, and source data
insufficient for either strategy; harvest; `ResolveResultRef` status tier;
snapshot compare.

Integration: create annotated PR → setup → selected task set only; verify
partial retry when required data is available; verify full fallback when
required results are missing/unreadable or the partial subgraph is empty;
verify terminal setup failure when neither strategy is constructible; UID
mismatch. Phase 2 integration should also test snapshot signature/provenance
validation, source UID and plan-digest mismatch, tenant binding, expiry,
unauthorized access, and large-result references.

E2E: flaky failure; matrix; `onError: continue`; RWO busy wait, deadline, and
failure behavior.

Chaos: source deleted after create; PVC deleted after setup; TaskRun pruned
before harvest.

### Infrastructure Needed

None beyond `tektoncd/pipeline` (and later optional Results).

### Upgrade and Migration Strategy

No migration of existing objects. New optional annotations/status. No `/retry`
endpoint to version.

## References

**Related TEPs:**
- TEP-0191: Resume a PipelineRun from a Task (resume vs retry)
- [TEP-0193: Tekton Artifacts OCI Storage](https://github.com/vdemeester/tektoncd-community/blob/e108293167b42e6f2f36b764ae67df9469c0c452/teps/0193-tekton-artifacts-oci-storage.md) (future storage integration)

---

