# TradingChassis Core Runtime

Evidence-based engineering and learning record for
[`TradingChassis/core-runtime`](https://github.com/TradingChassis/core-runtime)
(`tradingchassis-core-runtime` / import `core_runtime`).

This file is the canonical portfolio analysis record for this project. The
repository remains the technical source of truth. This file is not a CV,
README rewrite, or product page.

Repository analyzed: local clone `main` at **`a083981`** (2026-05-19).
Tag: **`v0.1.0`** (`d23d934`, 2026-02-18). Package version `0.1.0`.
License: MIT. Git authorship: **`bxvtr`** (14 commits on `main`).
Remote: `https://github.com/TradingChassis/core-runtime.git`.

**Working tree note:** the clone had uncommitted edits to `README.md` and
a Core git pin bump in `requirements*.txt`. This record uses **committed
HEAD** unless explicitly labeled otherwise.

How to read status labels:

| Label | Meaning in this file |
| --- | --- |
| Implemented | Present in `core_runtime/` or in-repo Argo/workflows |
| Tested | Covered by `tests/` |
| Documented local smoke path | Runnable command and fixtures are documented; **no committed run evidence or CI smoke job** |
| Locally usable | Config + synthetic NPZ + CLI exist |
| Cluster-capable | Manifests/workflows exist; not proven by committed run logs |
| Runtime-owned | This package |
| Core-provided | `tradingchassis_core` APIs consumed here |
| Infrastructure-provided | Cluster/PVC/IAM assumed from `TradingChassis/infrastructure` (not this repo) |
| Transitional compatibility path | Explicit shim / snapshot bookkeeping |
| Deferred | README + tests + design note; not shipped |
| Legacy | README banner from `#13` |
| Documented but not implemented | Docs without code |
| Inferred | Reasonable from multiple artifacts |
| Uncertain | Insufficient evidence |

Source-of-truth: **implementation and tests win.** The **legacy banner
wins** over present-tense packaging copy (`pyproject.toml` still
describes a “Kubernetes orchestration layer”).

---

## Project Status and Historical Context

**Status: Legacy / Architectural Exploration.**

The README opens with:

> This repository is no longer part of the active direction of
> TradingChassis. … TradingChassis has since pivoted away from
> implementing a custom trading engine.

That is **normative**. Runtime/orchestration code here is real
engineering evidence. It is **not** the current TradingChassis product
runtime, not a production trading Kubernetes platform, and not a claim
that the custom-engine direction continued.

The public GitHub repository is still described with present-tense
runtime/orchestration language. Treat that repository description like
the README body: **the explicit legacy banner wins**.

`TradingChassis/core` owns canonical events and decision semantics.
`TradingChassis/infrastructure` owns cluster provisioning. This repo
**consumes** both conceptually; it does not implement either.

Git: `#13` (`a083981`, 2026-05-19) is the legacy banner (README,
`pyproject.toml`, requirements pin lines). Last substantial runtime
code: `#12` (`f1175f2`, 2026-05-16) — align local backtest with clean
Core and stabilize the run loop. The clone’s working tree also had
**uncommitted** `requirements*.txt` Core-pin edits (`7be0826` →
`7c01945`) and README wording changes. Those are **not** HEAD.

Evidence:

```text
- README.md (status banner)
- git show a083981
- git show f1175f2
```

---

## Project Overview

**Status:** Implemented orchestration prototype around Core.
Locally usable with synthetic data. Cluster-capable via Argo templates.
Legacy as product direction.

Distribution **`tradingchassis-core-runtime` 0.1.0**, import
`core_runtime`. Python `>=3.11`. **No console scripts** — entrypoints
are `python -m …`. **No publish workflow.** Runtime deps in
`pyproject.toml` (ranges): `hftbacktest>=2,<3`, `mlflow>=3,<4`,
`oci>=2,<3`, `prometheus-client>=0.24,<1`. **`tradingchassis-core` is
not a pyproject dependency**; it is compiled in via
`scripts/compile-requirements.sh` + `.env`
`TRADINGCHASSIS_CORE_COMMIT`.

Committed `requirements.txt` pin (HEAD):

- `hftbacktest==2.4.4`
- `tradingchassis-core @ git+https://github.com/TradingChassis/core.git@7be082622bbe31aad81f6265b62429538e1f80f4`
  (Core `#13` contract-hardening commit, **before** Core’s own legacy
  banner `7c01945`)

**What this package does:** load JSON config, drive
**hftbacktest** as a simulation engine, map venue snapshots to
**canonical Core events**, call Core step APIs, dispatch
`dispatchable_intents` back into hftbacktest, write local artifacts,
and optionally plan/fan-out sweeps with OCI upload + MLflow metadata
on cluster.

**What it does not:** implement Core semantics, provision Kubernetes,
or persist/replay a canonical Event Stream.

Evidence:

```text
- pyproject.toml
- requirements.txt
- scripts/compile-requirements.sh
- core_runtime/local/backtest.py
```

---

## What I Built

Authorship on `main` is `bxvtr`. This provides strong evidence that I
was the primary author and maintainer of this runtime/orchestration layer,
its tests, and supporting workflow definitions before marking the
architecture legacy alongside Core.

### Runtime-owned — Implemented

1. Local CLI `python -m core_runtime.local.backtest`.
2. `HftStrategyRunner` wait-loop: venue adapter `wait_next` →
   hftbacktest `wait_next_feed` → canonical events → Core →
   execution adapter.
3. `EventStreamCursor` allocating `ProcessingPosition.index`.
4. Venue + execution adapters over hftbacktest (not a custom matcher).
5. Debug strategy (`DebugStrategyV1`) as a Core evaluator shim.
6. Experiment planner, sweep-context JSON, sweep worker
   (download → run → metadata → OCI put → cleanup).
7. `OCIObjectStorageS3Shim` (OCI SDK, boto3-**shaped** API).
8. MLflow **segment** logger (params/metrics/tags only).
9. Multi-stage Dockerfile (build runs `check.sh`; runtime non-root).
10. Argo WorkflowTemplates + GitHub launcher Workflows + self-hosted
    Actions that `kubectl apply` / submit.
11. pip-tools compilation with Core git SHA from `.env`.
12. Runtime tests (characterization, boundary guards, cursor, mapper).

### Core-provided (consumed, not authored here)

`run_core_step`, optional `run_core_wakeup_step`,
`process_event_entry`, `EventStreamEntry`, `ProcessingPosition`,
`StrategyState`, `RiskEngine` / `RiskConfig`, `ExecutionControl`,
`CoreConfiguration`, `EventBus` / `LoggingEventSink`, event types
constructed by the runner (`FileRecorderSink` is Runtime-owned).

### Infrastructure-provided (not this repo)

Cluster, Argo CD install, PVC `scratch-pvc`, OCI IAM for Instance
Principals, MLflow/Prometheus services named in templates.

### Deferred (explicit)

Runtime `FillEvent` ingress, `ExecutionFeedbackRecordSource`, Event
Stream persistence/replay, `ProcessingContext`.

Evidence:

```text
- core_runtime/backtest/engine/strategy_runner.py
- core_runtime/backtest/runtime/run_sweep.py
- tests/runtime/test_adapter_boundary_guards.py
- docs/venue-adapter-abstraction-design-v1.md
- README.md (Deferred capabilities)
```

---

## Core vs Runtime Ownership Boundary

| Concern | Owner |
| --- | --- |
| Canonical models, reducers, policy, Execution Control **semantics** | Core |
| Constructing `MarketEvent` / `OrderSubmittedEvent` / `ControlTimeEvent` / `OrderExecutionFeedbackEvent` | **Runtime** |
| `ProcessingPosition` allocation / commit | **Runtime** (`EventStreamCursor`) |
| Simulation matching, latency, queue, partial fills | **hftbacktest** |
| Dispatch of Core `dispatchable_intents` | Runtime execution adapter → hftbacktest |
| Artifact files | Runtime → local disk or OCI Object Storage |
| Experiment metadata (cluster) | Runtime → MLflow tracking API |
| Cluster/PVC/IAM | infrastructure repo |

The default `HftBacktestEngine` path constructs
`HftStrategyRunner` **without** `enable_core_step_wakeup_collapse`
(that flag is **False**). Wakeup collapse exists for tests / optional
construction, not the default local/Argo loop.

Evidence:

```text
- core_runtime/backtest/engine/strategy_runner.py
- core_runtime/backtest/engine/hft_engine.py
- core_runtime/backtest/engine/event_stream_cursor.py
```

---

## Local hftbacktest Runtime

**Status:** Implemented. A local smoke path is documented and supported
by config plus synthetic fixtures, but this audit found **no committed
run evidence and no CI smoke job**. Tested at adapter/runner unit level
with mocks — **no pytest invokes the full NPZ smoke**.

Command (repo root, cwd-relative paths):

```bash
python -m core_runtime.local.backtest --config core_runtime/local/bt_config_local.json
```

Flow:

```text
JSON config (cwd-relative paths)
  → HftEngineConfig + StrategyConfig + RiskConfig + CoreConfiguration
  → hftbacktest BacktestAsset / ROIVectorMarketDepthBacktest / Recorder
  → HftStrategyRunner loop
       venue.wait_next → hbt.wait_next_feed(include_order_resp, timeout_ns)
       rc 1: end
       rc 2: depth snapshot → MarketEvent(book snapshot) → run_core_step
       rc 3: state_values → OrderExecutionFeedbackEvent; orders discarded
       due ControlSchedulingObligation → ControlTimeEvent → run_core_step
       dispatchable_intents → HftBacktestExecutionAdapter
       successful new → OrderSubmittedEvent via process_event_entry
  → stats.npz (hftbacktest Recorder.to_npz)
  → events.jsonl (Core EventBus + Runtime FileRecorderSink; log, not replay)
```

Default outputs: `.runtime/local/results/stats.npz`,
`.runtime/local/results/events.jsonl` (directory gitignored).

Inputs: synthetic `tests/data/parts/part-00{0,1,2}.npz`. Instrument
`BTC_USDC-PERPETUAL`. Fees **0**. Constant latency 10ms entry/response.
`max_steps` 5e6. Strategy `DebugStrategyV1` (spread/qty/post_only).

**hftbacktest** (external): market replay, depth, matching, latency,
queue model, partial fill, `submit_*` / `cancel` / `modify`, stats NPZ.

**Runtime:** config, wait-loop, book snapshot → `MarketEvent`
(`event_type="book"`, `book_type="snapshot"`, price currency
`"UNKNOWN"`), Core call, intent dispatch, JSONL bus. Canonical market
path is **book snapshots**, not trade prints (`last_trades_capacity`
configures hftbacktest; it is not mapped to a Core trade event).

This is **not** “I built a backtesting engine.” Accurate: **integrated
hftbacktest behind a Runtime adapter that feeds canonical events into
Core.**

Correctness boundary: synthetic data, zero fees, debug strategy, no
FillEvent ingress. A successful smoke does **not** prove realistic
fills, live parity, or profitability.

Evidence:

```text
- core_runtime/local/backtest.py
- core_runtime/local/bt_config_local.json
- core_runtime/backtest/engine/hft_engine.py
- core_runtime/backtest/adapters/venue.py
- core_runtime/backtest/adapters/execution.py
- tests/data/parts/
```

---

## Canonical Event Integration and Compatibility Paths

**Status:** Partial canonical path Implemented + Tested. Fill ingress
**Deferred** (tests **forbid** constructing `FillEvent` in adapters).

README splits these as **canonical runtime paths**
(`MarketEvent`, `OrderSubmittedEvent`, `ControlTimeEvent`) vs
**runtime-local compatibility handling**
(`OrderExecutionFeedbackEvent` from account aggregates). Code
constructs all four.

### Events Runtime actually injects

| Event | When | Notes |
| --- | --- | --- |
| `MarketEvent` | hftbacktest `rc == 2` | Book snapshot from ROI depth ticks; `ts_ns_exch = ts_ns_local = sim_now`; no trade-print event |
| `OrderSubmittedEvent` | successful **new** dispatch | Runtime-side submit after `apply_intents`; **not** an exchange ACK |
| `ControlTimeEvent` | pending Core `ControlSchedulingObligation` (or scalar deadline shim) due vs sim clock | `reason="scheduled_control_recheck"`; clock is simulator `current_timestamp` |
| `OrderExecutionFeedbackEvent` | `rc == 3` | Account aggregates from `state_values` (position, balance, fee, volume, trades); **raw `orders` snapshot discarded** (`_ = orders`) |

`process_event_entry` is used for `OrderSubmittedEvent`; market /
feedback / control use `run_core_step`. Optional
`run_core_wakeup_step` exists behind
`enable_core_step_wakeup_collapse` (tests; **not** default engine).

Client order IDs: Core `client_order_id` string → adapter
`_to_i64_order_id` (numeric parse or blake2b → i64) for hftbacktest.

### Ordering

Runtime owns order. `EventStreamCursor.attempt_position()` /
`commit_success()` assigns strictly sequential `ProcessingPosition.index`
starting at 0. Core only checks monotonicity of what it is given.

Timestamps for market events equal the simulator clock (not a separate
exchange vs local pair).

### Transitional compatibility

- Raw venue order snapshots stay **runtime-local**; Core sees
  account-level feedback only.
- `_LegacyOnFeedStrategyEvaluator` / `_LegacyOnOrderUpdateStrategyEvaluator`
  adapt `Strategy.on_feed` / `on_order_update` to Core evaluator
  protocols.
- Successful `new` skips `mark_intent_sent` to avoid stale inflight
  after submitted reduction (tested).
- Scalar-only control deadline without a full obligation object is
  commented as transitional.

### Deferred (not shipped)

- Runtime `FillEvent` ingress
- `ExecutionFeedbackRecordSource` (not even a Protocol class)
- Replay / Event Stream persistence (`FileRecorderSink` JSONL is a
  **log sink**, not replay)
- `ProcessingContext`

**Do not claim full order/fill lifecycle integration.**

Evidence:

```text
- core_runtime/backtest/engine/strategy_runner.py
- core_runtime/backtest/engine/event_stream_cursor.py
- tests/runtime/test_strategy_runner_canonical_market_adoption.py
- tests/runtime/test_adapter_boundary_guards.py
- tests/runtime/test_hftbacktest_execution_feedback_probe.py
- README.md (canonical vs deferred tables)
```

---

## Experiment Planning and Sweep Execution

**Status:** Implemented. Tested for Core config presence on plan/sweep
paths. Grid overlay application has a **gap**.

Entrypoints:

| Module | Role |
| --- | --- |
| `core_runtime.backtest.runtime.entrypoint` | `--plan` (summary only) or `--run` (emit sweep JSONs) |
| `core_runtime.backtest.runtime.run_sweep` | One context: materialize, run, metadata, persist, cleanup |
| `segment_finalize_entrypoint` | MLflow segment metadata |
| `experiment_finalize_entrypoint` | Prometheus (not Core) |

IDs: `experiment_id` = config `id`; `segment_id` =
`segment_{index:04d}`; `sweep_id` = `sweep_{index:04d}` from a
**sorted-key cartesian product** (`itertools.product`); engine run id
`{experiment_id}__{segment_id}__{sweep_id}`. Emit file
`{segment_id}__{sweep_id}.json`. Empty grid still emits `sweep_0000`.

`SweepContext`: identity, dataset `file_keys`, `parameters`, scratch
paths. Scratch layout:
`{scratch_root}/{experiment_id}/{segment_id}/data|results/{sweep_id}`.
Worker `chdir`s into the segment dir for the engine run.

**Sweep overlay gap:** emit stores grid values under
`parameters["sweep"]` but `run_sweep.main()` builds
`HftEngineConfig` / strategy / risk from **base**
`parameters["engine"|"strategy"|"risk"]` only. The grid is in metadata
JSON; it is **not** applied to the live engine config. Treat parameter
sweeps as **partially implemented**.

Worker failure model: exception → no persist, re-raise; `finally`
**always** deletes sweep scratch results (`keep_scratch=False`), even
on failure. Download skip if `_READY` exists (idempotent materialize).
Persist requires local `_DONE`, then uploads **every file** in the
results dir (`iterdir` order; `_DONE` is not guaranteed last on OCI).
Do **not** call this an atomic object-store commit.

Argo segment finalize **hardcodes** `failed-sweeps: "0"` and
`completed-sweeps` = `expected-sweeps`. Failed sweep pods fail the
DAG rather than incrementing that counter.

`SweepMetadataWriter` records `GIT_COMMIT` / `GIT_DIRTY` /
`GIT_BRANCH` env, package version, Python/platform, `IMAGE_TAG`.
Fallback distribution name is `"tradingchassis-core"` if pyproject
cannot be read — a provenance footgun. **Core git SHA is not a
dedicated metadata field.**

Evidence:

```text
- core_runtime/backtest/runtime/entrypoint.py
- core_runtime/backtest/runtime/context.py
- core_runtime/backtest/runtime/run_sweep.py
- core_runtime/backtest/orchestrator/planner.py
- tests/runtime/test_runtime_core_configuration_integration.py
```

---

## Artifact Storage and OCI Integration

**Status:** Implemented adapter. Cluster path defaults to Instance
Principal. No adapter-level retries or checksums.

Class `OCIObjectStorageS3Shim`: **OCI Object Storage SDK**, not the
S3-compat HTTP endpoint. Method names are boto3-like (`put_object`,
`list_objects`, `get_object`, `download_to_file` with 8 MiB chunks).

Auth:

- default `auth_mode="instance_principal"` —
  `InstancePrincipalsSecurityTokenSigner` (OCI compute / IAM).
- `api_key` — `~/.oci/config` style; `core_runtime/local/oci.config.example`
  exists for local setups. Sweep worker **does not** pass `auth_mode`
  → Instance Principal.

Callers hardcode `region="eu-frankfurt-1"`, bucket **`data`**, prefix
`backtests/{experiment_id}/{segment_id}/{sweep_id}/` with
`stats.npz`, `events.jsonl`, `sweep_metadata.json`, `_DONE`.

No retry loop, no content hash verify, no multipart API, no
If-None-Match / overwrite guard. ETag captured best-effort on put.
**Do not** inflate SDK defaults into a reliability product.
Reruns of the same experiment/segment/sweep IDs will **overwrite**
object keys.

Infrastructure owns bucket/IAM/PVC; this repo **integrates**.

Evidence:

```text
- core_runtime/backtest/io/s3_adapter.py
- core_runtime/backtest/runtime/run_sweep.py
- core_runtime/local/oci.config.example
```

---

## MLflow Tracking Boundary

**Status:** Implemented metadata logger. **No artifact APIs.**

`MlflowSegmentLogger`: `MLFLOW_TRACKING_URI`, experiment =
`experiment_id`, run name = `segment_id`. Params:
`expected_sweeps` / `completed_sweeps` / `failed_sweeps`. Metric:
`duration_seconds`. Tags: `status`, ids. **No**
`log_artifact`. Documented as best-effort; segment finalizer swallows
exceptions.

**OCI = files. MLflow = metadata only.** That split is Implemented.

Evidence:

```text
- core_runtime/backtest/runtime/mlflow_segment_logger.py
- README.md (Backtest storage vs MLflow tracking)
```

---

## Runtime Packaging and Dependency Pinning

**Status:** Implemented compilation + image. Reproducibility is
**layered and incomplete**.

`compile-requirements.sh`: requires `.env`
`TRADINGCHASSIS_CORE_COMMIT`, writes a temp git URL
(`git+https://github.com/TradingChassis/core.git@$SHA`), `piptools
compile` → `requirements.txt` / `requirements-dev.txt`. Transitive
pins via pip-tools. **Not** hash-pinning (`--generate-hashes`
unused). **No SHA-format validation** in the compile script (any
non-empty env value is passed through). `.env.example` shows a
truncated placeholder, not a real pin.

Hex SHA validation **does** exist in the Kaniko template, for
**`core_runtime_commit`** (this repo’s image SHA), not for the Core
library pin.

Dockerfile:

- build: `python:3.11.14-slim-trixie`, install requirements, `pip
  install .`, run `./check.sh`
- runtime: same base, `appuser` non-root, copy `/install` only
- `ARG/ENV GIT_COMMIT`, `GIT_BRANCH`, `GIT_DIRTY`

Kaniko template: `--target=runtime`, hex SHA guard on
`core_runtime_commit`, tags `{repo}:{branch}`, `{repo}:{sha}`,
`:latest` if `git_branch=main`. SHA tags are **convention**, not
registry immutability or digest pin. No SBOM/signing in-repo.

`mypy` `ignore_errors = true` — CI still *runs* mypy but does not
enforce types.

Evidence:

```text
- scripts/compile-requirements.sh
- Dockerfile
- argo/templates/workflowtemplate-build-push-ghcr.yaml
- pyproject.toml (mypy ignore_errors)
```

---

## Argo / Kubernetes Orchestration

**Status:** Cluster-capable manifests + GitHub submit path.
**Cluster-validated: Uncertain** (no committed workflow logs/reports).

Split matches README:

| Path | Role |
| --- | --- |
| `argo/templates/*.yaml` | Reusable `WorkflowTemplate` (Argo UI) |
| `.github/argo-launchers/*.yaml` | One-off `Workflow` wrappers + `envsubst` |
| `.github/workflows/argo-build-and-backtest.yaml` | Self-hosted `microk8s kubectl` apply/submit |

Namespaces: `main` → `prod`, else `dev`. “Production-like,” not
production SRE. **No GitHub Environment protection rules** in this
workflow. `push` only lists `main`; `workflow_dispatch` can still
target other refs. The `else → dev` branch is therefore mostly for
manual dispatch.

`backtest-fanout`: DAG plan → `withParam` sweeps (`parallelism: 3`),
DAG `parallelism: 4`, PVC **`scratch-pvc`** (pre-provisioned; not
created here) at `/mnt/scratch`, in-image `bt_config_argo.json`
(`engine.data_files` is `null` until the worker injects downloaded
paths), MLflow/Prometheus in-cluster DNS names, `runAsUser 1000`.
Template `image_tag` **defaults to `main`** (mutable). README
recommends SHA for prod-like runs.

GitHub launcher `run-backtest.yaml` passes `IMAGE_TAG`. On `push` to
`main`, the workflow sets `IMAGE_TAG="${GITHUB_SHA}"` after the
Kaniko build succeeds (poll: 60 × 10s). Dispatch runs the backtest
only if `run_backtest=true`. That is **tag convention**, not a
registry digest pin.

GitHub job creates SA `argo-workflow-sa` and docker-registry secret
`ghcr-secret` from `secrets.GHCR_TOKEN`. Uses `sudo microk8s kubectl`
on a **self-hosted** runner. **No kubeconfig file is committed.**
**Does not** Terraform a cluster.

No SBOM, image signing, or Argo lint job in this repo.

Evidence:

```text
- argo/templates/workflowtemplate-build-push-ghcr.yaml
- argo/templates/workflowtemplate-backtest-fanout.yaml
- .github/argo-launchers/run-build.yaml
- .github/argo-launchers/run-backtest.yaml
- .github/workflows/argo-build-and-backtest.yaml
```

---

## Testing and Validation

**Status:** Tested (unit/characterization). Local smoke **manual**.
CI: lint + pytest + wheel reinstall. **No** Docker/Argo lint job on
`tests.yaml`.

73 `def test_*` functions across 11 runtime files plus `tests/test_dummy.py`.
`[tool.importlinter]` has **no** `forbidden_modules` contracts — the
linter still runs. Strongest:

| File | Locks |
| --- | --- |
| `test_adapter_boundary_guards.py` | Adapters must not import `FillEvent` / `process_event_entry` / mutate `StrategyState`; no `ExecutionFeedbackRecordSource` |
| `test_strategy_runner_canonical_market_adoption.py` | Market/control/rc3 paths, submitted vs inflight, no-progress stop |
| `test_event_stream_cursor.py` | attempt/commit index contract |
| `test_core_configuration_mapper.py` | JSON `core` → `CoreConfiguration` |
| `test_runtime_core_configuration_integration.py` | local/plan/sweep require `core` block |
| `test_hftbacktest_execution_adapter_characterization.py` | submit/modify/cancel mapping; not FillEvent |
| `test_hftbacktest_execution_feedback_probe.py` | hftbacktest surfaces exist but are **ineligible** as feedback record source |
| `test_hftbacktest_venue_adapter_recording.py` | `IndexError` swallow on record |
| `test_debug_strategy_state_api_compat.py` | debug strategy helpers |
| `test_import_compatibility_shim.py` | import identity only |

`tests.yaml`: Python 3.11, `pip install -e .` + `requirements-dev.txt`,
`./scripts/check.sh` (import-linter, ruff, mypy-with-`ignore_errors`,
pytest), `python -m build`, re-check on wheel. **Does not** run local
backtest CLI, compile requirements, or build the runtime Dockerfile.
**No committed Argo run logs, screenshots, or workflow output
artifacts.** Cluster-capable ≠ cluster-validated.

Evidence:

```text
- tests/runtime/
- scripts/check.sh
- .github/workflows/tests.yaml
```

---

## Reproducibility and Provenance Boundaries

Pinned / addressable:

- Runtime git SHA (image tag / `GIT_COMMIT` env if build args set)
- Core git SHA **in compiled `requirements.txt` inside the image**
- pip-tools transitive pins (no hashes)
- Experiment JSON + `SweepContext` identity/paths
- `sweep_metadata.json`: experiment/segment/sweep ids, UTC times,
  `parameters` (includes unused `sweep` overlay), `GIT_*`, image tag,
  Python version, artifact filenames

Not pinned / not proven:

- Base image digest
- Dataset content identity beyond object keys
- Live application of sweep grid to engine config
- Registry digest pin / immutable tags
- Core SHA as a first-class metadata field (only via installed dist /
  requirements in the image)
- Runtime **determinism** of hftbacktest (no seed story in this repo)

Prefer: **reproducible configuration and provenance hooks**, not
“deterministic distributed backtesting.” `CHANGELOG.md` claims
“Deterministic Docker runtime image” and “Immutable runtime
environment definition”; implementation is a tagged image plus
pip-tools pins, not digest-locked immutability. Treat those
changelog lines as **Level 2 overclaim**.

### Failure, idempotency, and secrets

| Case | Behavior |
| --- | --- |
| Missing sweep context file | `FileNotFoundError` |
| Missing config sections | `ValueError` / mapper errors |
| Download / backtest / upload exception | no persist; re-raise; scratch still cleaned |
| `_READY` present | skip re-download |
| `_DONE` missing | persist refuses |
| Same sweep IDs rerun | OCI `put_object` overwrites; MLflow `start_run` creates a **new** run (not idempotent) |
| MLflow failure | best-effort; segment finalizer swallows |

Secrets **submitted** by this repo’s workflows/templates (not owned
as a secrets platform):

- `secrets.GHCR_TOKEN` → K8s `ghcr-secret` for Kaniko
- OCI Instance Principal (cluster default) or local `api_key` config
  file
- `MLFLOW_TRACKING_URI` (in-cluster URL in the template; no password
  in this tree)
- Core git dependency is a **public** GitHub URL at a SHA (no token
  in compile script)

### Relationship to `TradingChassis/infrastructure`

This repo **names** `scratch-pvc`, `mlflow-svc.mlflow.svc.cluster.local`,
`prometheus-pushgateway.monitoring.svc.cluster.local`, Instance
Principal, and `microk8s kubectl`. Provisioning those services,
PVCs, IAM policies, and the GitOps cluster is **infrastructure-repo
work**. Core Runtime only consumes them.

Evidence:

```text
- core_runtime/backtest/runtime/run_sweep.py (SweepMetadataWriter)
- scripts/compile-requirements.sh
- Dockerfile
- CHANGELOG.md
- TradingChassis/infrastructure (separate inventory)
```

---

## Engineering Decisions and Trade-offs

1. **Runtime consumes Core** instead of reimplementing reducers. Cost:
   version pin + evaluator shims.
2. **hftbacktest as matcher**, Runtime as adapter. Cost: FillEvent never
   wired; lifecycle is incomplete by design of this slice.
3. **Explicit deferred list + tests that forbid FillEvent** — honest
   transitional architecture.
4. **Account-level feedback, snapshots stay local** — avoids pushing
   venue rows into Core. Cost: Core does not see per-order venue state.
5. **OCI files vs MLflow metadata** — clean split. Cost: no experiment
   artifact UI in MLflow.
6. **Instance Principal in cluster, API key template locally.**
7. **Argo templates vs GitHub launchers** — UI vs CI submit.
8. **SHA image tags as convention**, `:latest` on main — operator
   discipline required.
9. **cwd-relative JSON paths** — local smoke assumes repo root.
10. **Mark the engine runtime legacy** after building it — documented
    pivot, not silent abandonment of the code.

---

## Key Technical Learnings

- Semantic Core and execution Runtime can be separate packages with a
  thin event-mapping loop.
- Simulation engines (hftbacktest) are **providers**; adapters are the
  integration work.
- Compatibility shims should be named as such (legacy evaluators,
  unused order snapshots).
- Experiment context JSON is provenance; unused nested `sweep`
  parameters are a lesson in overlay bugs.
- Artifact object storage and tracking databases solve different
  problems.
- Cluster fanout via Argo `withParam` is orchestration, not a scale
  proof.
- Commit-pinned Core + pip-tools is stronger than floating
  `pyproject` ranges, weaker than hash+digest pins.
- cwd/path assumptions leak into “reproducible” local configs.
- Application manifests ≠ infrastructure ownership.
- A custom runtime can be **implemented and then retired** as product
  direction without erasing the engineering evidence.

---

## Historical Evolution and Pivot

| When | What |
| --- | --- |
| `bd2b3d9` 2026-02-18 | First commit |
| `#1`–`#2` | Tests/deploy workflow fixes; tag `v0.1.0` |
| `#3`–`#9` | Devcontainer, package data, CI, Argo build-then-backtest, rename/cleanup |
| `#10` 2026-05-05 | Integrate canonical Core processing |
| `#11` | Correct Core SHA |
| `#12` 2026-05-16 | Align local backtest with clean Core; large `strategy_runner` change |
| `#13` 2026-05-19 | **Legacy banner** (docs + pin files) |

Technical arc: **orchestration + hftbacktest → canonical Core loop →
explicit legacy status.** Same calendar week as Core `#14`.

Evidence:

```text
- git log --oneline
- CHANGELOG.md (initial 0.1.0 themes; does not narrate the pivot)
```

---

## Evidence Index

| Area | Paths |
| --- | --- |
| Local runner | `core_runtime/local/backtest.py`, `bt_config_local.json` |
| Core loop | `core_runtime/backtest/engine/strategy_runner.py` |
| hftbacktest | `hft_engine.py`, `adapters/venue.py`, `adapters/execution.py` |
| Cursor | `engine/event_stream_cursor.py` |
| Sweeps | `runtime/entrypoint.py`, `run_sweep.py`, `context.py` |
| OCI | `backtest/io/s3_adapter.py` |
| MLflow | `runtime/mlflow_segment_logger.py` |
| Pinning | `scripts/compile-requirements.sh`, `requirements.txt` |
| Image | `Dockerfile` |
| Argo | `argo/templates/*`, `.github/argo-launchers/*` |
| GHA | `.github/workflows/tests.yaml`, `argo-build-and-backtest.yaml` |
| Tests | `tests/runtime/` |
| Deferred design | `docs/venue-adapter-abstraction-design-v1.md` |
| Legacy | `README.md` banner, `a083981` |
| Secrets | `.github/workflows/argo-build-and-backtest.yaml`, `core_runtime/local/oci.config.example` |

Web search (2026-08-20): used only to confirm public GitHub org/repo
presence. [TradingChassis](https://github.com/TradingChassis) lists
`core-runtime` as a **legacy** orchestration layer (0 stars, last
update 2026-05-19). No extra implementation evidence beyond the
clone.

---

## Limitations and Non-Claims

Do not use this repository as evidence for:

- Current TradingChassis production runtime.
- Building hftbacktest or a matching engine.
- Production distributed backtesting / high-throughput cluster.
- Full Event Stream persistence, replay, or `ProcessingContext`.
- Full order/fill integration (`FillEvent` deferred; tests forbid it).
- Automated local smoke or committed live Argo success.
- Immutable image bytes (SHA tag ≠ digest pin).
- Sweep grid actually driving engine parameters.
- Enforcing mypy (`ignore_errors = true`).
- Owning Terraform/Ansible/Argo CD/PVC provisioning.
- Atomic OCI uploads or idempotent MLflow experiment storage.
- Trade-print / fill-complete market integration (book snapshots
  only on the canonical market path).

Keep:

> Implemented a legacy runtime/orchestration prototype around
> TradingChassis Core, including a local hftbacktest-backed backtest
> path, sweep execution, OCI artifact handling, MLflow metadata
> logging, and Argo-based cluster workflow definitions.

---

## Derived Defensible Experience Statements

Valid only with the limits above.

- Built a Python runtime that **consumes** `tradingchassis_core`
  (`run_core_step`, policy/apply contexts) while owning config, the
  hftbacktest wait-loop, and `ProcessingPosition` allocation.
- Mapped simulator book snapshots to canonical `MarketEvent`s, runtime
  submits to `OrderSubmittedEvent`, sim clock + Core obligations to
  `ControlTimeEvent`, and account `state_values` to
  `OrderExecutionFeedbackEvent` — without FillEvent ingress.
- Integrated **hftbacktest** as the simulation engine behind venue and
  execution adapters (submit/cancel/modify, latency/queue/partial-fill
  configuration).
- Planned experiments into `SweepContext` JSON and implemented a worker
  that downloads inputs from OCI, runs one backtest, writes
  `sweep_metadata.json`, uploads artifacts, and cleans scratch.
- Separated **OCI Object Storage files** from **MLflow tracking
  metadata**.
- Packaged a multi-stage image, pip-tools-compiled deps with a **Core
  git SHA**, and Argo WorkflowTemplates plus GitHub launchers for
  build/fanout on a self-hosted `microk8s` path.
- Documented and test-locked deferred capabilities, then marked the
  repository **legacy** when TradingChassis left the custom-engine
  direction.

Those statements are invalid if rewritten as “I built a production
Kubernetes trading platform,” “I built hftbacktest,” “full fill
lifecycle,” or “current TradingChassis runtime.”
