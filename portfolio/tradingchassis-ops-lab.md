# TradingChassis Ops Lab

Evidence-based engineering and learning record for
[`TradingChassis/tradingchassis-ops-lab`](https://github.com/TradingChassis/tradingchassis-ops-lab).

This file is a source of truth for later GitHub-profile, CV, LinkedIn, and
interview use. It is not a CV, README rewrite, or marketing summary.

Repository analyzed: local clone at `0.8.0` / `main` (`98a95d7`, 2026-06-21).
Package version: `0.8.0` (`pyproject.toml`, `src/tradingchassis_ops_lab/__init__.py`).
Git authorship in this repository: all sampled commits are under `bxvtr`.
LICENSE copyright: `tradingeng@protonmail.com`.

How to read status labels:

| Label | Meaning in this file |
| --- | --- |
| Implemented | Present in Python, tests, configs, CLI, or Compose |
| Partially implemented | Present, with an explicit boundary or unused placeholder |
| Historical / placeholder | File exists but is not the active path |
| Planned / deferred | Roadmap or `NotImplementedError` only |
| Documented but stale | Docs/comments that contradict current code |
| Inferred | Reasonable conclusion from multiple artifacts |
| Uncertain | Insufficient evidence |

Source-of-truth rule: implementation wins over documentation.
This project is **not** a strategy-performance, PnL, live-trading, or
exchange-connectivity system. Those non-claims are restated at the end.

---

## Project Overview

**Status:** Implemented as a local-first operations lab around NautilusTrader.

TradingChassis Ops Lab is a Python CLI (`tc`) that models **spec-driven runs**,
writes a stable per-run artifact tree, and layers file-based operational
controls (kill switch, reconciliation, failure drills, connectivity
readiness/probe, backtest-vs-paper evidence) on top of those artifacts.

It uses **NautilusTrader as an external backtest engine** (`nautilus_trader==1.227.0`).
The authored work is the run contract, artifact/journal/report layer, safety
and drill tooling, and artifact-backed metrics — not a custom matching engine.

What it is:

- A local lab for **operational evidence** of backtest and paper *lifecycles*.
- An **artifact-first** interface: later commands read JSON/JSONL/Markdown
  on disk rather than inspecting a live trading process.
- A bounded demo of SRE-style ideas (reconciliation, kill switch, drills,
  dashboards) without claiming production trading safety.

What it is not (implemented tree agrees with README/scope/limitations):

- Profitable strategy, alpha, PnL, Sharpe, or returns analysis.
- Live, testnet, or real-exchange connectivity.
- Real order submission, fills, account, balance, or position management.
- Low-latency or production-grade trading infrastructure.
- A custom trading engine.

Evidence:

```text
- README.md
- docs/scope.md
- docs/limitations.md
- docs/adr/0001-use-nautilus-as-engine.md
- pyproject.toml
- src/tradingchassis_ops_lab/cli.py
```

---

## What I Built

Attribution in this repository is strong: Git history is a single-author
sequence from `a1b0bd2` (2026-05-20) through `98a95d7` (2026-06-21), tags
`0.1.0`–`0.8.0`. That supports **authored / designed / implemented / tested /
documented** for the Python package and ops configs. It does not mean
NautilusTrader, Prometheus, or Grafana were written here.

### Authored in this repo — Implemented

1. **Run contract** — Pydantic `RunSpec` (`spec_version: v1`), YAML load,
   unknown-field forbid, SHA-256 of canonical JSON (`config_sha256`).
2. **CLI** — Typer app `tc` with groups: `spec`, `run`, `data`, `metrics`,
   `kill`, `reconcile`, `drill`, `connectivity`, `evidence`.
3. **Backtest orchestration** — Load prepared 1m OHLCV, run Nautilus
   `BacktestEngine` with one built-in scenario, persist lifecycle artifacts.
4. **Built-in scenario** — `OpsSmokeDemoStrategy` (`ops_smoke_demo`): bar
   counting and a deterministic flag at bar index 5; **no orders**.
5. **Paper lifecycle skeleton** — Three synthetic heartbeats, no market feed,
   no Nautilus `engine.run()`, gated by file-based kill switch.
6. **Dataset prepare/fingerprint** — Copy fixture `btcusdt-sample`, content
   hashes; fingerprint is **not** a runtime gate.
7. **Artifact-backed metrics** — Prometheus text from stored files; HTTP
   `/metrics` server; optional one-shot export.
8. **Local observability stack** — Docker Compose Prometheus + Grafana,
   provisioned dashboard reading those metrics.
9. **Kill switch** — Per-run JSON state + JSONL events; paper start blocks
   when `active`.
10. **Reconciliation** — Expected vs observed JSON (position, open orders,
    freshness); writes `reconciliation_result.json`; non-zero exit on
    mismatch/unknown.
11. **Failure drills** — Three deterministic drills over fixtures/artifacts;
    mismatch drill exits 1 **by design**.
12. **Connectivity readiness** — Env **name** presence only; no network.
13. **Connectivity probe** — Read-only HTTP GET, **loopback hosts only**.
14. **Evidence compare** — Artifact-to-artifact backtest vs paper comparison;
    known gaps include “PnL not available”.
15. **Docs/runbooks** — MkDocs site, ADRs, failure-mode inventory, GitHub
    Pages workflow.

### Dependency-provided (not authored here)

- NautilusTrader `BacktestEngine`, bar wrangling, `TestInstrumentProvider`,
  `Strategy` base class.
- Prometheus and Grafana container images and upstream query language.
- Typer, Pydantic, PyYAML, pandas.

### Placeholders still in the tree — Historical / deferred

```text
src/tradingchassis_ops_lab/engines/nautilus/paper.py
  run_paper_placeholder() → NotImplementedError
src/tradingchassis_ops_lab/safety/guards.py
  evaluate_guards_placeholder() → NotImplementedError
```

Active paper behavior lives in `runs/paper.py`, not the engine adapter.
Do not treat those placeholders as implemented paper trading or a risk engine.

Evidence:

```text
- git log (bxvtr; tags 0.1.0–0.8.0)
- src/tradingchassis_ops_lab/**
- tests/unit/**
- docs/adr/0001-use-nautilus-as-engine.md
```

---

## Run and Artifact Model

**Status:** Implemented.

The operational contract is: **a run is a directory of files**, not a
long-lived process. Commands either create that directory or mutate/read
files inside it (or under `artifacts/evidence/`).

### RunSpec

Validated YAML → `RunSpec` (`extra="forbid"`):

| Field | Role |
| --- | --- |
| `spec_version` | Literal `v1` |
| `run_id` | Directory name under `artifacts/runs/` |
| `mode` | `backtest` or `paper` |
| `engine` | Literal `nautilus` (label; paper does not execute the engine) |
| `venue` / `instrument` | Labels; backtest additionally requires `binance` + `BTCUSDT` |
| `strategy.name` / `version` | Scenario identity, not a plugin path |
| `data.dataset` / `fingerprint` | Dataset name; fingerprint is metadata only |
| `risk.profile` | Placeholder string |
| `observability.*` | Recorded in metadata; **do not disable** artifact writes |
| `connectivity_readiness` | Optional local preflight contract |

`tc spec validate` loads the spec and prints `run_id` / `mode` / `engine`.
`tc run init` copies the spec, writes `metadata.json` and the first journal
event. Re-init of an existing `run_id` fails (`RunArtifactsAlreadyExistError`).

`config_sha256` is SHA-256 over canonical JSON (`sort_keys`, compact
separators). That hash is **deterministic for a given spec**. Timestamps in
metadata/journal are **not** deterministic (`datetime.now(UTC)`).

**Stale comment:** `ConnectivityReadinessSpec` in `runs/spec.py` still says
the block “does not perform … environment validation.” `0.5.0` added
`evaluate_connectivity_readiness`, which **does** check env name presence.
Treat the class docstring as stale; the evaluator is implemented.

### Canonical per-run layout

```text
artifacts/runs/<run_id>/
  run_spec.yaml
  metadata.json
  journal.jsonl
  metrics.json          # after backtest/paper (not after init alone)
  report.md             # after backtest/paper
  connectivity_readiness.json   # optional
  connectivity_probe.json       # optional
  reconciliation_result.json    # optional
  drills/<drill_name>.json      # optional
```

Cross-run evidence (does **not** mutate the compared run dirs):

```text
artifacts/evidence/<backtest_run_id>__<paper_run_id>/
  backtest_vs_paper_evidence.json
  backtest_vs_paper_evidence.md
```

Kill-switch runtime state is **outside** the run dir:

```text
runtime/kill_switch/<run_id>.state.json
runtime/kill_switch/<run_id>.events.jsonl
```

Generated `data/`, `artifacts/runs/`, `artifacts/evidence/`, and `runtime/`
are gitignored. Curated samples live under `reports/sample/`.

### Lifecycle events (journal)

JSONL, one object per line, `sort_keys=True`. Typical backtest sequence:
`run_started` → `backtest_started` → `backtest_completed` → `run_completed`
(or `run_failed`). Paper: `run_started` → `paper_safety_checked` → either
`paper_safety_blocked` or heartbeats/`paper_completed` → `run_completed`.

Status values observed in code: `initialized`, `running`, `completed`,
`failed`, `safety_blocked`.

Later tools (metrics, evidence, drills, connectivity) **patch** metadata,
append journal lines, and optionally splice Markdown sections between HTML
comment markers. That is the artifact-first pattern: operators reconstruct
what happened from files after the process exits.

Evidence:

```text
- src/tradingchassis_ops_lab/runs/spec.py
- src/tradingchassis_ops_lab/runs/artifacts.py
- src/tradingchassis_ops_lab/runs/hashing.py
- src/tradingchassis_ops_lab/runs/metadata.py
- src/tradingchassis_ops_lab/runs/journal.py
- docs/run-model.md
- examples/configs/btcusdt_backtest.yaml
- examples/configs/btcusdt_paper.yaml
```

---

## Backtest and Paper Lifecycle

**Status:** Implemented, with sharply different meanings for each mode.

### Backtest — Nautilus smoke over candles

**Orchestration authored here:** `runs/backtest.py` +
`engines/nautilus/backtest.py`.

**Engine capability (NautilusTrader):** `BacktestEngine`, venue/account,
instrument, bar wrangling, strategy callbacks, `engine.run()`.

Flow:

1. Require `mode=backtest`; initialize artifacts; status `running`.
2. Load `data/datasets/<dataset>/candles_1m.csv` (must be prepared).
3. Resolve instrument via `TestInstrumentProvider.btcusdt_binance()` —
   **only** `venue=binance` and `instrument=BTCUSDT`.
4. Build `BarType` `{instrument}-1-MINUTE-LAST-EXTERNAL`.
5. Register **one** built-in strategy from a hardcoded factory
   (`ops_smoke_demo` only; unknown names raise `UnknownBacktestScenarioError`).
6. `BacktestEngineConfig(run_analysis=False)` — analysis/PnL reports from
   the engine are not the product.
7. Add venue (NETTING, CASH, 1_000_000 USDT starting balance), instrument,
   strategy, bars; `engine.run()`; `engine.dispose()`.
8. Read **operational counters** from `OpsSmokeDemoStrategy.counters()`,
   not from Nautilus performance reports.
9. Write `metrics.json`, `report.md`, complete metadata/journal.

`ops_smoke_demo` (`engines/nautilus/strategies/ops_smoke_demo.py`):

- Subscribes to bars; increments `bars_seen`.
- At `action_bar_index` (default 5) sets `deterministic_action_triggered`.
- `on_order_submitted` / `on_order_filled` exist for future counting.
- **Does not call submit/cancel.** Expected: `orders_submitted = 0`,
  `fills_count = 0`.

This is engine **integration and lifecycle wrapping**, not “built a trading
engine” and not a strategy research harness.

### Paper — bounded synthetic skeleton

**Implemented in** `runs/paper.py`. Does **not** call
`run_nautilus_backtest_smoke` and does **not** implement
`engines/nautilus/paper.py`.

Behavior:

- Requires `mode=paper`.
- Reads kill-switch snapshot **before** creating artifacts.
- Writes metadata with `is_placeholder: true`, `connectivity: none`,
  `lifecycle: paper_skeleton`.
- If kill switch is `active`: status `safety_blocked`, zero heartbeats,
  blocked metrics/report; **does not start the skeleton**.
- Else: three `paper_heartbeat` journal events (`synthetic: true`),
  `heartbeat_count: 3`, `synthetic_duration_seconds: 3` in metrics
  (**constants**, not a real 3-second sleep of market time).
- `engine_executed: false`. `engine` field remains the spec label
  `nautilus`.

Venue `binance_testnet` in the example spec is a **label**. It does not
open a testnet socket.

Evidence:

```text
- src/tradingchassis_ops_lab/runs/backtest.py
- src/tradingchassis_ops_lab/runs/paper.py
- src/tradingchassis_ops_lab/engines/nautilus/backtest.py
- src/tradingchassis_ops_lab/engines/nautilus/config.py
- src/tradingchassis_ops_lab/engines/nautilus/strategies/ops_smoke_demo.py
- src/tradingchassis_ops_lab/engines/nautilus/paper.py  # unused placeholder
- tests/unit/test_runs_paper_safety.py
```

---

## Data Preparation and Reproducibility

**Status:** Implemented for one fixture dataset; fingerprint **not** enforced
at run time.

- `tc data prepare --dataset btcusdt-sample` copies
  `fixtures/datasets/btcusdt-sample/candles_1m.csv` → `data/datasets/...`.
- Only `btcusdt-sample` is supported.
- `tc data fingerprint` hashes each file (SHA-256 + size), then hashes a
  canonical JSON of `{dataset, algorithm, files}` to `dataset_sha256`.
- Output: `data/fingerprints/btcusdt-sample.fingerprint.json`.
- Example RunSpecs set `data.fingerprint: placeholder`. Backtest/paper
  **do not** compare that field to the fingerprint file (documented in
  `RunSpec` / README / roadmap gap: “fingerprint linkage is manual”).

What is actually reproducible:

| Piece | Deterministic? |
| --- | --- |
| `config_sha256` for identical spec | Yes |
| Dataset file hashes | Yes (content-based) |
| JSON dumps (`indent=2, sort_keys=True`) | Yes for payload shape |
| Journal/metadata timestamps | No (wall clock) |
| Kill-switch `event_id` | No (`uuid4`) |
| Probe `latency_ms` | No (measured) |
| Engine `engine_duration_ms` | No (measured) |
| `ops_smoke_demo` counters on same candles | Intended yes (bar index 5) |

Rerunning the same `run_id` is refused; reproducibility means **same spec +
same dataset → comparable artifacts**, not overwrite-in-place.

Evidence:

```text
- src/tradingchassis_ops_lab/data/prepare.py
- src/tradingchassis_ops_lab/data/fingerprint.py
- fixtures/datasets/btcusdt-sample/
- tests/unit/test_data_prepare.py
- tests/unit/test_data_fingerprint.py
- tests/unit/test_runs_hashing.py
```

---

## Operational Evidence and Reconciliation

**Status:** Implemented. This is **not** strategy performance.

### `tc evidence compare`

Reads two run directories; writes under `artifacts/evidence/`.
Path-safety: run IDs validated; resolved paths must stay under
`--artifacts-root`.

Core required files: `metadata.json`, `metrics.json`, `journal.jsonl`,
`report.md`. Missing any → `comparison_status: missing_artifacts` and empty
`compared_fields`.

If modes are not backtest vs paper → `incompatible_runs`.
Otherwise → `differences_expected` (mode differences are **expected**, not
a failed A/B of PnL).

Compared areas include engine/instrument labels, venue (contextual;
mismatch allowed), scenario vs paper strategy metadata, config hashes
(mismatch expected across mode-specific specs), `engine_executed` /
`is_placeholder` (mode-specific), artifact presence, shared journal event
names, paper safety state (gap on backtest), optional readiness/probe
(future_candidate), bars vs heartbeats (mode-specific).

Hard-coded `known_gaps` include `pnl_not_available`,
`returns_sharpe_not_available`, `paper_no_engine_execution`,
`paper_no_market_data`, `fill_quality_not_available_no_fills`,
`slippage_not_available_no_orders`, `orderbook_not_available_candle_only`.
`non_goals`: alpha, live paper trading, order execution, pnl, returns,
sharpe, strategy optimization.

The compare function is **pure with respect to run dirs** (read-only).
Timestamps in the evidence payload use `created_at_utc` (clock).

### `tc reconcile check`

File-based expected vs observed JSON fixtures (not a live broker snapshot).

Checks:

1. **Position** — symbol, side (`long|short|flat`), qty and avg price as
   `Decimal` (string fields, normalized).
2. **Open orders** — keyed by `order_id`; missing/extra IDs and field deltas.
3. **Freshness** — age of position/orders timestamps vs `max_age_seconds`;
   stale → **warning** (still `matched: true` for that check); missing max
   age → **unknown**.

Roll-up: `ok` < `warning` < `unknown` < `mismatch`. CLI exits 1 if status
is `mismatch` or `unknown`. Result written to
`artifacts/runs/<run_id>/reconciliation_result.json`; journal event
`reconciliation_checked` if `journal.jsonl` exists.

This models **invariant checking on documented state files**, not exchange
reconciliation.

Evidence:

```text
- src/tradingchassis_ops_lab/evidence/compare.py
- src/tradingchassis_ops_lab/evidence/write.py
- src/tradingchassis_ops_lab/reconciliation/checks.py
- examples/reconciliation/*.json
- tests/unit/test_evidence_compare.py
- tests/unit/test_reconciliation.py
- docs/backtest-vs-paper.md
- docs/runbooks/evidence-compare.md
- docs/runbooks/reconciliation-mismatch.md
```

---

## Failure Modes and Drills

**Status:** Implemented as **local simulated drills**, not production incidents.

`docs/failure-modes.md` (0.8.0) inventories local failure states: trigger,
exit code, artifact, journal, metric, dashboard, recovery. That document is
the contract; drills below are the executable subset.

| Command | What it does | Exit | Outcome field |
| --- | --- | --- | --- |
| `tc drill stale-market-data` | Reconciliation vs `observed_stale_warning.json` | 0 | `expected_warning` |
| `tc drill reconciliation-mismatch` | Reconciliation vs `observed_position_mismatch.json` | **1 by design** | `expected_mismatch` |
| `tc drill restart-recovery` | Checks metadata + `run_spec.yaml` exist; **no process restart** | 0 | `simulated_recovery_ok` |

Fixtures are rewritten to the CLI `run_id` in a temp dir, then
`run_reconciliation_check` is invoked. Reports go to
`artifacts/runs/<run_id>/drills/<name>.json`. Journal:
`failure_drill_executed`. Limitations listed in every report:
local file-based only; no exchange; no cancel/flatten/restart orchestration.

Grafana panels `Failure Drill Last Pass` / `Failure Drill Outcome` /
`Reconciliation Status` are empty until those JSON files exist — they are
**not** live incident telemetry.

Deferred (roadmap / limitations): missing-update drill, rate-limit drill,
stale-orderbook drill, `tc run verify` artifact linter, Alertmanager.

Evidence:

```text
- src/tradingchassis_ops_lab/drills/executor.py
- src/tradingchassis_ops_lab/cli.py (drill_* commands)
- tests/unit/test_drills.py
- docs/failure-modes.md
- docs/runbooks/*.md
```

---

## Runtime Safety

**Status:** Implemented as a **local file-based kill switch**. Not an exchange
emergency stop.

Storage:

- `runtime/kill_switch/<run_id>.state.json` — `active` or `cleared`.
- `runtime/kill_switch/<run_id>.events.jsonl` — `kill_activated` /
  `kill_cleared` with reason, actor, `uuid4` event id.
- Optional append of the same event object onto the run `journal.jsonl` if
  that file already exists.
- `tc kill activate|status|clear`; metadata patch
  `update_artifact_metadata_safety_snapshot`.

Paper gate (`run_paper_lifecycle`): if state is `active`, lifecycle does not
emit heartbeats; artifacts still exist with `safety_blocked`. Backtest is
**not** gated by the kill switch.

What it does **not** do (code and docs agree):

- Cancel orders at an exchange.
- Flatten positions.
- Halt Nautilus mid-`engine.run()` (backtest is already a batch job).
- Provide a production risk engine (`guards.py` is an unused placeholder).

Fail-closed property that **does** exist: paper will not enter the synthetic
heartbeat path while the file says `active`. Fail-open relative to trading:
there are no real orders to cancel.

Evidence:

```text
- src/tradingchassis_ops_lab/safety/kill_switch.py
- src/tradingchassis_ops_lab/runs/paper.py
- tests/unit/test_kill_switch.py
- tests/unit/test_runs_paper_safety.py
- docs/runbooks/safety-gate.md
```

---

## Connectivity Readiness and Probe Boundaries

**Status:** Implemented as **local preflight**, not venue connectivity.

### `tc connectivity readiness`

Requires an existing run directory (`tc run init` or a completed run).

- If the spec has no `connectivity_readiness` or `enabled: false` →
  `state=disabled`, `probe_performed=false`,
  `probe_deferred_reason=local_only_no_network`.
- If enabled: compare `required_env` / `optional_env` **names** against
  `os.environ` (non-empty after strip). Missing required →
  `missing_credentials`; all required present → `configured`.
- Artifacts list **names** in `present_env` / `missing_env`. Values are not
  written. Metrics are documented as not exposing env names.
- **No sockets.** CLI prints `probe_performed=false`.

Example paper spec requires `TRADINGCHASSIS_PAPER_API_KEY` and
`TRADINGCHASSIS_PAPER_API_SECRET` as placeholders. Presence ≠ valid keys
and ≠ Binance testnet login.

### `tc connectivity probe`

- URL must be `http://` only; no userinfo, query, or fragment.
- Host must be `localhost`, `127.0.0.1`, or `::1`.
- Method: GET. Default timeout 1000 ms.
- States: `probe_ok`, `probe_http_error`, `probe_timeout`,
  `probe_unreachable`, `probe_unknown`.
- `response_body_stored: false`; body is not persisted.
- **Command exit 0** after a completed probe even for timeout/unreachable
  (outcome is in the artifact). Invalid URL or I/O → exit 1.

This is a deliberate engineering limit: practice HTTP preflight and artifact
recording without opening a path to real venues.

Evidence:

```text
- src/tradingchassis_ops_lab/connectivity/readiness.py
- src/tradingchassis_ops_lab/connectivity/probe.py
- tests/unit/test_connectivity_readiness.py
- tests/unit/test_connectivity_probe.py
- docs/runbooks/connectivity-probe-failed.md
```

---

## Observability

**Status:** Implemented as **artifact-backed derived metrics** plus a local
scrape stack. Not live trading telemetry.

### Metric source

`observability/metrics.py` loads `metadata.json`, `metrics.json`, optional
journal, reconciliation, readiness, probe, `drills/*.json`, and evidence
JSON. `tc metrics serve` re-renders Prometheus text on each GET `/metrics`
(stdlib `ThreadingHTTPServer`). `tc metrics export` is a one-shot dump;
it still requires `metrics.json` (init + readiness alone is not enough).

Malformed drill/evidence files are skipped with comment lines (lenient),
unlike required run files which error on export.

### Naming (selected)

Per-run gauges/counters from artifacts include run info, duration,
backtest candle/bar/scenario counters (including
`orders_submitted_total` which is **zero** for `ops_smoke_demo`), paper
heartbeats, journal event totals, kill-switch state, readiness/probe,
reconciliation, drill pass/outcome encodings.

Aggregate evidence metrics (need `--evidence-root`): status totals, known
gaps total, compared fields, shared journal events.

Grafana dashboard **TradingChassis Ops Lab Run Observability** panels
include Run Info, durations, kill switch, readiness, probe state/latency,
evidence status/gaps, bars vs heartbeats, journal events, reconciliation,
drill pass/outcome. Compose binds `127.0.0.1` only; Prometheus scrapes
`TC_METRICS_TARGET` (default `host.docker.internal:8000`). Images are
`:latest` (not pinned).

Empty Grafana panels usually mean missing artifacts, not a downed exchange.

Evidence:

```text
- src/tradingchassis_ops_lab/observability/metrics.py
- src/tradingchassis_ops_lab/observability/serve.py
- deploy/observability/docker-compose.yml
- dashboards/grafana/tradingchassis-ops-lab-run-observability.json
- tests/unit/test_observability_metrics.py
- tests/unit/test_observability_serve.py
- tests/unit/test_observability_stack_configs.py
```

---

## Testing and Validation

**Status:** Implemented locally; **GitHub Actions does not run pytest**.

`scripts/check.sh`: `ruff format --check`, `ruff check`, `python -m pytest`.
Python `>=3.12`. Dev extra: pytest 9.0.3, ruff 0.15.13. No mypy in
`pyproject.toml` (a `.mypy_cache` directory exists locally; type-check is
not a declared package script).

Unit tests (under `tests/unit/`) cover spec, hashing, artifacts, journal,
metadata, paper safety, data prepare/fingerprint, CLI, kill switch,
reconciliation, drills, evidence, metrics/serve/stack configs, connectivity
readiness/probe, package/devcontainer integration. `tests/integration/`
is an empty package.

What tests lock as **operational contracts** (examples):

- Unknown scenario name fails closed.
- Paper blocks when kill switch is active.
- Probe rejects non-loopback URLs.
- Evidence compare treats PnL as a known gap / non-goal.
- Mismatch drill is expected to fail the reconciliation check.
- Metrics renderer reads files, not a live Nautilus process.

CI: `.github/workflows/docs-pages.yml` — MkDocs `build --strict` on PR;
deploy Pages on `main`. **No CI job for ruff/pytest.** Quality bar for
code is contributor-local (`CONTRIBUTING.md`).

Dev Container: Python 3.12 image, docker-outside-of-docker, ports 8000/9090/3000,
`pip install -e ".[dev]"`.

Evidence:

```text
- scripts/check.sh
- tests/unit/*.py
- .github/workflows/docs-pages.yml
- CONTRIBUTING.md
- .devcontainer/devcontainer.json
```

---

## Engineering Decisions and Trade-offs

Visible in the tree, not slogans.

1. **NautilusTrader instead of a custom engine** (ADR 0001).
   Finite scope; effort goes to ops contracts. Cost: paper still carries
   `engine: nautilus` as a label without executing the engine.

2. **Smoke scenario instead of a research strategy.**
   Rename from `toy_mean_reversion` to `ops_smoke_demo` (0.4.0) to avoid
   implying alpha. Cost: almost no strategy surface; plugin loading deferred.

3. **`run_analysis=False`.**
   Avoid treating Nautilus analysis as the lab’s output. Cost: no engine
   performance report even for curiosity.

4. **Paper as synthetic heartbeats.**
   Demonstrates lifecycle, artifacts, and safety gate without a data feed.
   Cost: evidence compare must encode mode differences as expected gaps.

5. **Artifacts as the interface between tools.**
   Metrics, drills, evidence, Grafana all consume files. Cost: no true
   streaming telemetry; dashboards lag until files exist.

6. **File-based kill switch.**
   Auditable, testable, no broker API. Cost: not cancel/flatten; easy to
   over-claim in interviews if scope is omitted.

7. **Loopback-only probe / env-name readiness.**
   Practices connectivity *contracts* without credentials or venue risk.
   Cost: does not prove Binance reachability.

8. **Reconciliation on fixtures.**
   Decimal-safe, explicit severities, drillable. Cost: not live positions.

9. **Mismatch drill exits 1.**
   Failure is a first-class, inspectable outcome. Cost: automation must
   treat that exit as success-of-the-drill, not a broken CLI.

10. **Observability before Kubernetes** (roadmap decision rule).
    Local Compose first. K8s/GitOps is explicitly deferred.

11. **Init refuses existing `run_id`.**
    Avoids silent overwrite. Cost: operators must choose new IDs or delete
    dirs; no in-place rerun.

12. **Docs CI but not test CI.**
    Pages stay buildable. Cost: pytest regressions are not gated on GitHub.

---

## Reliability and Operations Engineering

**Status:** Implemented as **lab-scale** reliability patterns.

What this repo actually demonstrates:

- Explicit success/failure/`safety_blocked` states instead of “process exited 0
  so trading worked”.
- Journals as append-only operational history.
- Reconciliation with ordered severities and fail-the-command on mismatch.
- Drills that produce artifacts operators can inspect after a non-zero exit.
- Kill switch that fails closed for the **paper skeleton start**.
- Bounded waits/timeouts on the probe only (not a distributed system).
- Runbooks mapping failure mode → artifact → metric → recovery.
- Path traversal checks on evidence output IDs.

What it does **not** demonstrate: HA, multi-node failover, broker disconnect
handling, order-path kill switches, or on-call production trading.

---

## Key Technical Learnings

Project-specific, from the implementation:

- **Artifacts can be the API.** Evidence compare, metrics serve, and drills
  never attach to a running strategy; they parse JSON. That forces schemas
  and makes “what happened” reviewable after the process is gone.
- **Mode differences are not bugs.** Backtest executes an engine; paper
  does not. Encoding that as `differences_expected` plus `known_gaps` is
  more honest than forcing a single “match” score.
- **Safety scope must be named.** A file that blocks `tc run paper` is a
  real control for that command and a non-control for exchanges. The
  interesting skill is keeping those sentences in the same design.
- **Failure can be a successful drill.** Exit 1 plus `outcome:
  expected_mismatch` is a contract, tested, and documented. Ambiguous
  “green” would hide the point.
- **Connectivity theater is a hazard.** Env presence and loopback GET are
  useful preflights; venue labels like `binance_testnet` are not
  connections. The code keeps `probe_performed=false` on readiness to
  make that hard to miss.
- **Determinism is partial.** Hashes and counters can be stable; clocks,
  UUIDs, and latencies cannot. Tests inject `now_utc` where it matters.
- **Reserved spec fields lie if you treat them as feature flags.**
  `observability.metrics: false` still gets `metrics.json` from lifecycle
  commands. Metadata ≠ runtime gating.
- **Wrapping an engine is not owning it.** Instrument provider, OMS types,
  and bar wrangling come from Nautilus. The lab owns scenario selection,
  counter export, and the file lifecycle around `engine.run()`.

---

## Historical Evolution

Extracted from tags and commit subjects; not a raw commit list.

| Version | When (tag era) | Technical step |
| --- | --- | --- |
| Scope docs | 2026-05-20 | Boundaries written before code (`a1b0bd2`) |
| `0.1.0` | 2026-05-21 | Spec, artifacts, dataset fingerprint, Nautilus smoke, paper skeleton, metrics export, kill switch, reconciliation, drills, MkDocs |
| `0.2.0` | 2026-05-22 | `metrics serve`, Compose Prometheus/Grafana |
| `0.3.0` | 2026-05-22 | Kill switch in paper path, safety metric/panel |
| `0.4.0` | 2026-05-26 | `ops_smoke_demo` contract; rename away from fake alpha name |
| `0.5.0` | 2026-05-28 | Readiness spec + env-name evaluation |
| `0.6.0` | 2026-05-28 | Loopback probe + probe metrics/panels |
| `0.7.0` | 2026-06-01 | Evidence compare + aggregate evidence metrics |
| `0.8.0` | 2026-06-21 | Failure-mode inventory, drill metrics, more runbooks |

Roadmap **next** (not implemented): Kubernetes/GitOps lab. Explicitly later:
strategy plugins, orderbook, historical import, runtime-state
reconciliation, alerting expansion, external read-only venue probe.

---

## Evidence Index

| Area | Primary paths |
| --- | --- |
| CLI surface | `src/tradingchassis_ops_lab/cli.py` |
| RunSpec / hashing | `runs/spec.py`, `runs/hashing.py` |
| Backtest wrap | `runs/backtest.py`, `engines/nautilus/backtest.py` |
| Scenario | `engines/nautilus/strategies/ops_smoke_demo.py` |
| Paper skeleton | `runs/paper.py` |
| Data | `data/prepare.py`, `data/fingerprint.py`, `fixtures/datasets/` |
| Evidence | `evidence/compare.py`, `evidence/write.py` |
| Reconciliation | `reconciliation/checks.py` |
| Drills | `drills/executor.py` |
| Kill switch | `safety/kill_switch.py` |
| Readiness / probe | `connectivity/readiness.py`, `connectivity/probe.py` |
| Metrics | `observability/metrics.py`, `observability/serve.py` |
| Compose / Grafana | `deploy/observability/`, `dashboards/grafana/` |
| Tests | `tests/unit/` |
| Local check script | `scripts/check.sh` |
| CI | `.github/workflows/docs-pages.yml` |
| Scope / limits | `docs/scope.md`, `docs/limitations.md`, `README.md` |
| Failure contract | `docs/failure-modes.md` |
| ADR engine choice | `docs/adr/0001-use-nautilus-as-engine.md` |
| Authorship | `git log`; tags `0.1.0`–`0.8.0` |

Git history used: `git log --oneline`, author stats, tags, first/last
commits. Not used as a substitute for reading the current tree.

Web search: not used.

---

## Limitations and Non-Claims

Do not use this repository as evidence for:

- A profitable, alpha-generating, or optimized trading strategy.
- Production, live, or testnet trading.
- Real exchange connectivity, signed endpoints, or credential validation.
- Real order execution, fills, positions, balances, or flattening.
- Low-latency or HFT systems.
- Production-grade safety or monitoring.
- That Prometheus/Grafana here is live process tracing of a trading engine.
- That GitHub Actions proved pytest green (it only builds docs).
- That `engine: nautilus` on paper runs means Nautilus executed.
- That `binance` / `binance_testnet` venue strings are live venues.
- That `data.fingerprint` or `observability.*` gate runtime behavior.
- That `safety/guards.py` or `engines/nautilus/paper.py` are active.

Implemented limits to keep visible:

- One dataset, one instrument, 1-minute candles only.
- One built-in scenario; no plugin strategies.
- Paper: three synthetic heartbeats; kill switch is a local file.
- Probe: loopback HTTP GET only.
- Reconciliation and drills: fixtures / artifact presence.
- Compose images unpinned (`:latest`).
- Kubernetes/GitOps deferred.

**Stale vs implementation:** `RunSpec.connectivity_readiness` class
docstring understates 0.5.0 env evaluation. `ansible`-style drift is
limited to that comment; README/CHANGELOG match the evaluator.

---

## Derived Defensible Experience Statements

These are bounded restatements of the evidence above, not CV polish.

- Designed a spec-driven run contract (Pydantic/YAML) with canonical config
  hashing and a refuse-overwrite artifact directory per `run_id`.
- Integrated NautilusTrader as a **smoke backtest engine** behind an
  authored lifecycle that records operational counters, not PnL.
- Implemented a built-in non-trading scenario (`ops_smoke_demo`) that
  counts bars and a deterministic action without submitting orders.
- Modelled paper mode as a bounded synthetic lifecycle with an explicit
  `engine_executed: false` and a file-based kill-switch start gate.
- Built artifact-first follow-on tools: reconciliation, drills, evidence
  compare, and Prometheus text derived from stored JSON/JSONL.
- Added loopback-only HTTP probe and env-name readiness checks that
  deliberately do not contact exchanges or store secrets.
- Encoded expected mode differences and known capability gaps in a
  backtest-vs-paper evidence schema instead of implying equivalence.
- Documented failure modes and runbooks that treat simulated drills as
  drills, not as production incidents.

Those statements remain invalid if the limits in the previous section are
dropped.
