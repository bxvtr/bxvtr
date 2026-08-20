# TradingChassis Core

Evidence-based engineering and learning record for
[`TradingChassis/core`](https://github.com/TradingChassis/core)
(`tradingchassis_core`).

This file is the canonical portfolio analysis record for this project. The
repository remains the technical source of truth. This file is not a CV,
README rewrite, or product page.

Repository analyzed: local clone `main` at `7c01945` (2026-05-19).
Tag: **`v0.1.0`** (`224b221`, 2026-02-17). Package version `0.1.0`.
License: MIT. Git authorship on this clone: **`bxvtr`** (14 commits on
`main`). Local `origin` URL is
`https://github.com/trading-engineering/trading-platform.git`;
`pyproject.toml` documents `https://github.com/TradingChassis/core`.

How to read status labels:

| Label | Meaning in this file |
| --- | --- |
| Implemented | Present in `tradingchassis_core/` |
| Tested | Covered by `tests/semantics/` |
| Partially implemented | Code exists with documented or observed gaps |
| Architecture exploration | Design that was implemented as a prototype, then retired as product direction |
| Legacy | Explicit README status from `#14` (2026-05-19) |
| Convenience implementation | Optional class implementing a protocol; not auto-wired |
| Extension point | Protocol / context the caller must supply |
| Runtime-owned / external | Explicitly outside this package |
| Documented but not implemented | README/docs claim without matching code |
| Planned | Roadmap / deferred cleanup |
| Inferred | Reasonable conclusion from multiple artifacts |
| Uncertain | Insufficient evidence |

Source-of-truth rule: **implementation and tests win over README body.**
The **legacy banner wins over present-tense product language** still
sitting below it in the README.

---

## Project Status and Historical Context

**Status: Legacy / Architectural Exploration.**

The README opens with:

> This repository is no longer part of the active direction of
> TradingChassis. … TradingChassis has since pivoted away from
> implementing a custom trading engine.

That statement is **normative for this portfolio file**. Do not treat
`tradingchassis_core` as the current TradingChassis product core, a
production trading engine, or live execution infrastructure.

What the tree still is:

- A **implemented and semantics-tested Python library** of deterministic
  Event-step decision semantics (reduction → strategy → candidates →
  policy → execution-control apply).
- Historical proof that a shared backtest/live **decision** API was
  designed, implemented, and then **retired as engine direction**.

What it is not:

- The active TradingChassis engine.
- A Venue adapter, Runtime, Kubernetes control plane, or OMS.
- Proof that backtest and live **fills** match.

Git: `#14` (`7c01945`, 2026-05-19) only adds the legacy banner (README +
`pyproject.toml`). Library code last moved in `#13` (`7be0826`,
2026-05-18). The pivot is **documented**, not reverse-engineered.

`TradingChassis/core-runtime` is a **separate** repository. This record
does not attribute Runtime work to Core. Dead-code notes that mention
Runtime callers are **in-repo documentation**, not an audit of that
other tree.

Evidence:

```text
- README.md (status banner)
- git show 7c01945
- git show 7be0826
```

---

## Project Overview

**Status:** Implemented prototype library. Legacy as product direction.
Tested at the semantics layer. Alpha classifier in packaging.

Installable package **`TradingChassis-core` 0.1.0**, import
`tradingchassis_core`. **Not** src-layout: package lives at repo root.
`requires-python >= 3.11`. Runtime dependency: **`pydantic>=2,<3` only**.
Dev extra: pytest, import-linter, ruff, mypy, build. Classifier:
Development Status :: 3 - Alpha. **No publish workflow** in
`.github/workflows` — do not claim PyPI.

**What Core does (code):** consume caller-supplied **canonical**
`EventStreamEntry` values, reduce `StrategyState`, optionally evaluate a
**Strategy protocol**, combine **Intents** into candidates, optionally
run **policy admission**, optionally **plan/apply Execution Control**,
and return a frozen `CoreStepResult`. It does **not** talk to exchanges.

**Canonical** here means: typed Pydantic/dataclass models Core is
willing to reduce. Runtime is assumed to **normalize** raw feeds into
those models. Core does not parse venue JSON or WebSockets.

Evidence:

```text
- pyproject.toml
- tradingchassis_core/__init__.py
- tradingchassis_core/core/domain/processing_step.py
```

---

## What I Built

Authorship on `main` is `bxvtr`. This provides strong evidence that I
was the primary author and maintainer of the package, its semantics tests,
and supporting documentation. It does not mean I currently operate
TradingChassis as this engine.

### Authored — Implemented

1. **Event-step pipeline** — `run_core_step` / wakeup APIs over
   `process_event_entry`.
2. **Canonical event set** — book `MarketEvent`, order submitted,
   cancel/reject/expire, fill, execution feedback, `ControlTimeEvent`.
3. **`ProcessingPosition` monotonic cursor** on positioned ingestion.
4. **Mutable `StrategyState` + snapshot `StrategyStateView`**
   (`MappingProxyType` + frozen sub-views).
5. **Intent model** — `NewOrderIntent` / `CancelOrderIntent` /
   `ReplaceOrderIntent` (venue-agnostic commands).
6. **Candidate combine** with typed dominance on `client_order_id`.
7. **Policy admission protocol** (`PolicyIntentEvaluator`) vs optional
   **`RiskEngine`** convenience implementation.
8. **Execution Control** queue / inflight / token-bucket apply;
   **rate-limit** `ControlSchedulingObligation` only.
9. **Semantics test suite** (12 files, 45 `test_*` functions) plus two
   runnable examples.
10. **CI** — `scripts/check.sh` (import-linter, ruff, mypy, pytest) on
    source and on a built wheel.

### Extension points (caller-supplied)

- `CoreStepStrategyEvaluator` / `CoreWakeupStrategyEvaluator`
- `PolicyIntentEvaluator` (may be `RiskEngine`)
- `ExecutionControl` instance
- `CoreConfiguration`, `EventBus` (`NullEventBus` for tests)

### Runtime-owned / external (not in this package)

Raw feeds, normalization, wall-clock scheduling loops, HTTP/WebSocket
dispatch, credentials, adapters, Kubernetes. Confirmed by **no**
network/file/env I/O in `tradingchassis_core/` (see Runtime boundary).

### Convenience vs mechanism

| Piece | Role |
| --- | --- |
| Policy admission helpers | Mechanism — wired when `CorePolicyAdmissionContext` is passed |
| `RiskEngine` | Convenience `PolicyIntentEvaluator` — **not** the default of `run_core_step` |
| Execution Control plan/apply | Mechanism — wired when apply context is passed |
| `ExecutionControl` | Default queue/rate/inflight object — instance still supplied by caller |
| `NullEventBus` | No-op bus |

Evidence:

```text
- tradingchassis_core/__init__.py
- docs/code-map/core-pipeline-map.md
- tests/semantics/
```

---

## Architecture and Ownership Boundaries

Implemented pipeline when the caller supplies strategy, policy, and
apply contexts (`activate_dispatchable_outputs=True`):

```text
Runtime canonical Event
    → EventStreamEntry + ProcessingPosition
    → process_event_entry / process_canonical_event   (state reduction)
    → Strategy evaluation (optional; StrategyStateView)
    → generated Intents
    → queued snapshot + candidate merge / dominance
    → Policy admission (generated only; queued passthrough)
    → Execution Control plan
    → Execution Control apply (queues, rate, inflight)
    → CoreStepResult.dispatchable_intents
      + optional ControlSchedulingObligation (rate-limit only)
    → Runtime-owned external dispatch / later ControlTimeEvent injection
```

If **no** policy context: reduction + strategy + candidates only;
`dispatchable_intents` stay empty; `core_step_decision` is `None`.

If policy is set but **no** apply context: policy runs; plan is
computed; apply does **not** run; result dispatchables stay empty.

Apply without policy **raises** `ValueError`.

Core **does not execute orders**. `dispatchable_intents` are outputs.
Venue serialization and APIs are Runtime/adapter-owned.

Evidence:

```text
- tradingchassis_core/core/domain/processing_step.py
- docs/code-map/core-pipeline-map.md
- README.md (Full pipeline)
```

---

## Canonical Event and State Model

**Status:** Implemented. Tested for book-only market, fills, terminals,
feedback vs inflight.

### What “canonical” means in code

A **closed set of types** `process_canonical_event` will reduce.
Unknown types → `TypeError`. `ControlSchedulingObligation` is
**non-canonical** (helper output, not stream input).

Core does **not** normalize venue payloads. **Runtime normalizes; Core
consumes canonical events.**

### Events actually reduced

| Type | Role |
| --- | --- |
| `MarketEvent` | Book BBO → market snapshot (mid = 0.5×(bid+ask) when both > 0) |
| `OrderSubmittedEvent` | Working order `submitted`; clear inflight |
| `OrderCanceledEvent` / `OrderRejectedEvent` / `OrderExpiredEvent` | Terminal: pop working, projection, clear inflight |
| `FillEvent` | Fills / cum qty / remaining; emit on `EventBus` |
| `OrderExecutionFeedbackEvent` | **Account only** (not orders/fills/inflight) |
| `ControlTimeEvent` | Advance local control timestamp only |

**No `OrderAcceptedEvent`.** No full order state machine (README
matches code).

**Trade-shaped `MarketEvent`:** schema allows `event_type="trade"`;
**reduction rejects** with `ValueError` (tested). Empty book / missing
book rejected. Positioned book path **requires**
`CoreConfiguration` instrument `tick_size`, `lot_size`,
`contract_size` all finite `> 0`.

### Processing order

`ProcessingPosition.index` (`>= 0`, frozen). Positioned processing
requires **strictly increasing** index (`>` last). Equal or decreasing
→ `ValueError`. Gaps are allowed (not required consecutive). Event
**timestamps are not** compared to the cursor. Direct
`process_canonical_event(..., position=None)` **bypasses** the cursor
(used as a primitive; `process_event_entry` always passes position).

Monotonicity is **Implemented**; **no dedicated test** named for
`Non-monotonic ProcessingPosition` was found.

### `StrategyState`

Mutable container (not a frozen dataclass): market, account, working
orders, fills (deque cap 10_000 on apply), fill cum qty, queued
intents, inflight, canonical order projections, last-sent intents,
rolling equity, last local timestamp, processing cursor.

### `StrategyStateView`

**API contract: read-only snapshot for strategy.** Implementation:
`MappingProxyType` over **frozen** `MarketStateView` /
`AccountStateView` / `WorkingOrderView`; fills via
`model_copy(deep=True)`. **Not** a `@dataclass(frozen=True)` wrapper
class. **Does not expose** `queued_intents`, `inflight`, or reducers
(tested).

Python can still mutate a copied Pydantic `FillEvent` (models use
`extra="forbid"`, not necessarily `frozen=True`); that copy is
**isolated** from `StrategyState`. This is **contract + snapshot**,
not a capability-safe sandbox.

Policy admission receives **`StrategyState`**, not the view — a
`PolicyIntentEvaluator` *can* mutate state. `RiskEngine` rolling-loss
can `popleft` `rolling_equity` (docs say side-effect-free; **code
wins** for that gate).

Evidence:

```text
- tradingchassis_core/core/domain/processing.py
- tradingchassis_core/core/domain/processing_order.py
- tradingchassis_core/core/domain/state.py
- tradingchassis_core/core/domain/types.py
- tests/semantics/test_market_event_contract.py
- tests/semantics/test_order_terminal_lifecycle.py
- tests/semantics/test_strategy_state_view_boundary.py
```

---

## Strategy and Intent Boundary

**Status:** Implemented protocols. Tested with synthetic evaluators.
No bundled production strategy.

Strategy is an **extension point**. One evaluator per `run_core_step`
(after **that** event is reduced) or one wakeup evaluator after **all**
batch entries are reduced.

Intents are **internal commands**, not venue order messages:

- Shared: `ts_ns_local`, `instrument`, `client_order_id`, optional
  correlation id.
- `NewOrderIntent`: side, `limit`/`market`, qty, price, TIF
  `{GTC, IOC, FOK, POST_ONLY}`.
- `CancelOrderIntent`: cancel only.
- `ReplaceOrderIntent`: limit-only qty/price.

Venue-specific serialization, client-order mapping, and HTTP/WS stay
**Runtime-owned**. `SlotKey` / `stable_slot_order_id` are exported
helpers (docs: also used outside Core; this file does not audit that).

Evidence:

```text
- tradingchassis_core/core/domain/processing_step.py (evaluator protocols)
- tradingchassis_core/core/domain/types.py
- examples/core_step_quickstart.py
```

---

## Candidate Combination, Dominance and Reconciliation

**Status:** Implemented. Tested in pipeline tests (same
`client_order_id` generated wins).

`CandidateIntentRecord` (frozen): `intent`, `origin` (`GENERATED` |
`QUEUED`), `logical_key`, `merge_index`, `priority`.

**Logical key:** `order:{client_order_id}` (instrument **not** in the
key).

**Combine** (`combine_candidate_intent_records`): **pure**; queued
first, then generated. Dominance on the same key:

```text
rank(cancel)=3 > rank(replace)=2 > rank(new)=1
higher rank wins
equal rank → later in merge order wins
```

Sort of survivors: `(priority, merge_index, logical_key)` with cancel
priority 0, replace 1, new 2.

This layer is **intent-vs-intent**, not OMS working-order
reconciliation.

**Execution Control apply** is a second, **intent-vs-queue/working**
layer (duplicate NEW, cancel of queued-only, replace noop, inflight
enqueue). Call that **sendability / queue reconciliation**, not a full
exchange order book reconcilor.

Evidence:

```text
- tradingchassis_core/core/domain/intent_combination.py
- tradingchassis_core/core/domain/candidate_intent.py
- tradingchassis_core/core/domain/execution_control_apply.py
- tests/semantics/test_core_pipeline_clean.py
```

---

## Policy Admission and Risk Engine

**Status:** Mechanism Implemented + Tested. `RiskEngine` is
**Convenience implementation**, Tested as an evaluator, **not** always
wired.

### Mechanism

`PolicyIntentEvaluator.evaluate_policy_intent(...) -> (bool, reason?)`.

`apply_policy_to_candidate_records`:

- **QUEUED** → passthrough (not re-checked).
- **GENERATED** → evaluator.
- Rejection records are **policy** rejections, distinct from
  `OrderRejectedEvent` (execution outcome injected later by Runtime).

`now_ts_ns_local` is **injected** (not `time.time()`).

### `RiskEngine` gates actually in code

Via `RiskPolicy` / `ExecutionConstraintsPolicy` (not an institutional
risk platform):

- `trading_enabled` (cancels still allowed when disabled)
- max drawdown / optional rolling loss
- tick/lot normalize for **checks** (dispatch still uses **original**
  intent — Implemented, easy to miss)
- min notional, POST_ONLY would-trade
- validate qty/price/instrument/ts / market mid for market orders
- max position, max single-order notional, max gross notional
- quote-count / quote-notional caps when configured

`RiskConfig.order_rate_limits` is **not** what Execution Control
consumes. Apply uses `max_orders_per_sec` / `max_cancels_per_sec` on
`CoreExecutionControlApplyContext`.

Evidence:

```text
- tradingchassis_core/core/domain/policy_risk_decision.py
- tradingchassis_core/core/risk/risk_engine.py
- tradingchassis_core/core/risk/risk_policy.py
- tests/semantics/test_risk_engine_policy_only.py
- tests/semantics/test_risk_engine_pipeline_integration.py
- tests/semantics/test_order_terminal_lifecycle.py (policy ≠ OrderRejectedEvent)
```

---

## Execution Control and Scheduling Obligations

**Status:** Implemented. Tested for rate-limit obligation vs inflight
non-obligation.

**What it is:** in-process **dispatch now / queue / defer** for
already-admitted intents. Token bucket on **injected**
`now_ts_ns_local`. Mutates Core queues and `ExecutionControl` bucket
state. **Does not** send bytes to a venue.

**Plan** is pure (concatenates accepted generated + queued passthrough).
**Apply** performs duplicate/inflight/rate/queue-local handling.

`CoreStepResult.dispatchable_intents` is populated only when apply ran
**and** `activate_dispatchable_outputs=True` (Runtime/core contract
test).

### Inflight

Gating keyed by instrument + client order id. Deferral **does not**
emit `ControlSchedulingObligation` by default (docs + tests). Inflight
is set by `mark_intent_sent`; the step pipeline **does not** call that
method — Runtime (or tests) must. README: not the default Runtime/Core
send contract; inflight tests **seed it**.

### Rate limit

Fixed token bucket in `ExecutionControl.consume_rate`. On block: build
`ControlSchedulingObligation(reason="rate_limit", due_ts_ns_local=wake_ts, ...)`.
**Not** adaptive 429 handling. **Not** a Core scheduler loop.

### `ControlSchedulingObligation` vs `ControlTimeEvent`

| | Obligation | `ControlTimeEvent` |
| --- | --- | --- |
| Canonical stream? | **No** | **Yes** |
| Produced by | Execution Control apply (rate-limit) | Runtime injection when/if it realizes a due time |
| Reducer | n/a | timestamp only |

Core does **not** sleep or `asyncio.sleep` until `due_ts_ns_local`.

Evidence:

```text
- tradingchassis_core/core/execution_control/execution_control.py
- tradingchassis_core/core/execution_control/types.py
- tradingchassis_core/core/domain/execution_control_apply.py
- docs/flows/control-time-and-scheduling.md
- tests/semantics/test_control_time_scheduling_semantics.py
- tests/semantics/test_runtime_core_contract.py
```

---

## Order Lifecycle and Feedback

**Status:** Partial lifecycle. Tested terminals + fill + feedback
split.

Supported **inbound** lifecycle events: submitted, canceled, rejected,
expired, fill. **Not** accepted. Working-order projection is a **thin
map**, not an OMS state machine.

`OrderRejectedEvent` ≠ policy rejection.

`OrderExecutionFeedbackEvent` updates account (inventory/balance/fees);
it does **not** clear inflight (tested).

Evidence:

```text
- tradingchassis_core/core/domain/processing.py
- tests/semantics/test_fill_event_reduction.py
- tests/semantics/test_order_terminal_lifecycle.py
- tests/semantics/test_runtime_core_contract.py
```

---

## Determinism and Wakeup Semantics

**Status:** Designed and implemented under **stated assumptions**.
**Tested** via ordered reduction and “evaluate once after batch”
tests. **Not** property-tested; **no** replay-hash / “run twice, hash
equal” test.

### When Core is deterministic

For the **same**:

- initial `StrategyState`
- ordered `EventStreamEntry` sequence
- `CoreConfiguration`
- injected `now_ts_ns_local` values
- strategy / policy / `ExecutionControl` objects that themselves are
  pure w.r.t. those inputs

the library path does **not** read `datetime.now`, `time.time`,
`random`, env, network, or filesystem.

Determinism is **not** “bit-identical to a live exchange.” It is
**same canonical stream + config + injected clocks → same Core
outputs**, modulo extension-point purity.

### What Core does **not** enforce

- Wall-clock-free **strategies** (protocol cannot stop `time.time()` in
  user code).
- Consecutive processing indexes.
- Timestamp vs index agreement.
- Identical fills vs production.

### Wakeup (Tested)

**Not** parallel event processing. `run_core_wakeup_reduction` reduces
entries **in order**, then **one** `CoreWakeupStrategyEvaluator`.
`run_core_wakeup_decision` does **one** candidate/policy/apply pass.
`run_core_wakeup_step` = reduction + decision.

`run_core_step` = one entry, one optional step evaluator, one decision
pass.

**Backtest/live parity:** Core **explores reducing decision-logic
drift** by sharing this API. The repo does **not** prove identical
exchange behavior, fills, or a realistic simulator.

Evidence:

```text
- tradingchassis_core/core/domain/processing_step.py
- tests/semantics/test_core_wakeup_final_state.py
- tests/semantics/test_core_pipeline_clean.py
```

I/O scan of `tradingchassis_core/`: no `datetime.now` / `time.time` /
HTTP / websocket. Clocks are event fields and injected `now_ts_ns_local`.
`LoggingEventSink` may log if registered (not on the default
`NullEventBus` path). `__version__` reads package metadata.

---

## Runtime / Venue Adapter Boundary

**Status:** Isolation Implemented in this tree (no venue modules).
import-linter contract name is “Core stays runtime-independent” but
`forbidden_modules = []` — **mechanical forbid list is empty**; purity
is the import graph of this package, not a populated lint denylist.

No exchange names, HTTP clients, or order-wire codecs under
`tradingchassis_core/`.

`EventBus` is **in-process fanout** (`emit` on fills), not the Event
Stream, not metrics SaaS.

Evidence:

```text
- pyproject.toml ([tool.importlinter.contracts])
- tradingchassis_core/core/events/event_bus.py
- tradingchassis_core/core/events/sinks/null_event_bus.py
```

---

## Testing and Verification

**Status:** Semantics suite Tested. Examples runnable; Risk Engine
example **is** invoked from tests. No Hypothesis. No live venue.

| File | What it locks |
| --- | --- |
| `test_public_api_clean.py` | Root `__all__` lock + docs mention |
| `test_core_pipeline_clean.py` | Step path, dominance, policy block, rate-limit obligation, book reduce |
| `test_core_wakeup_final_state.py` | Reduce-all-then-evaluate-once; wrapper ≡ split APIs |
| `test_market_event_contract.py` | Book ok; trade rejected at reduction/step/wakeup |
| `test_fill_event_reduction.py` | Fill reducer + evaluator sees reduced fill |
| `test_order_terminal_lifecycle.py` | Cancel/reject/expire; policy ≠ execution reject |
| `test_strategy_state_view_boundary.py` | View hides queue/inflight/reducers |
| `test_runtime_core_contract.py` | Feedback vs inflight; dispatch flag; obligation non-canonical |
| `test_control_time_scheduling_semantics.py` | Rate-limit obligation; inflight no obligation |
| `test_risk_engine_policy_only.py` | `RiskEngine.evaluate_policy_intent` accept |
| `test_risk_engine_pipeline_integration.py` | `RiskEngine` in `run_core_step` |
| `test_runnable_risk_engine_example.py` | `examples/core_step_with_risk_engine.py` exits 0 |

Fixtures are **synthetic** (tiny books, stub evaluators, allow-all
policies). Strong for **contracts**, not for market realism.

### CI (`scripts/check.sh` + `.github/workflows/tests.yaml`)

On push/PR to `main` and `workflow_dispatch`: Python 3.11, `pip install
-e ".[dev]"`, then import-linter, ruff, mypy, pytest; `python -m build`;
re-run `check.sh` against the wheel in a venv. **No Deribit/live
orders.**

Evidence:

```text
- tests/semantics/*.py
- scripts/check.sh
- .github/workflows/tests.yaml
```

---

## Engineering Decisions and Trade-offs

1. **Decision library vs engine product.** One pipeline for “same
   semantics in every runtime.” Later **retired** as TradingChassis
   engine direction — documented in `#14`.
2. **Canonical events, Runtime normalizes.** Keeps Core I/O-free. Cost:
   Core is useless without a honest Runtime.
3. **Intents, not venue orders.** Adapter mapping stays outside. Cost:
   two languages (intent vs wire order).
4. **Policy protocol vs `RiskEngine` convenience.** Avoids baking one
   risk theory into the kernel. Cost: easy to confuse mechanism with
   the built-in gates.
5. **Policy rejection vs `OrderRejectedEvent`.** Different layers,
   tested. Cost: operators must learn two “reject” words.
6. **Execution Control = sendability, not execution.** Rate-limit
   obligation instead of sleeping. Cost: Runtime must actually schedule
   `ControlTimeEvent`.
7. **Wakeup = ordered fold + one decision.** Avoids per-tick strategy
   storms. Cost: not a parallel actor model.
8. **Strategy view snapshot vs policy getting full state.** Strategy
   cannot see queues by default; policy can mutate (rolling equity).
9. **Narrow public `__all__` + U3 dead-code removal.** Public-surface
   tests lock the intended root exports. Cost: some helpers remain
   internal/deferred.
10. **Pydantic contracts, extra forbid.** Strict models. Cost: not all
    models are frozen; view isolation is snapshot-based.

---

## Key Technical Learnings

Only where this tree supports them:

- **Separating deterministic decisions from I/O** is a real boundary
  (no HTTP in Core) and still depends on extension-point discipline.
- **Canonical streams** are a type/reducer contract, not magic
  normalization.
- **Intent vs order** lets strategy stay venue-agnostic; someone else
  must still be correct on the wire.
- **Admission reject ≠ exchange reject.** Mixing them hides whether
  risk or the venue said no.
- **Dominance is a total order on intent families per
  `client_order_id`**, not a general conflict solver.
- **Rate-aware control can be an obligation record**, not a thread
  sleep, if the Runtime closes the loop with `ControlTimeEvent`.
- **Inflight is feedback-driven**; inventing a wake time would lie.
- **Shared Core APIs explore backtest/live logic reuse**; they do not
  equalize venues.
- **Building a custom trading engine is optional.** This repo’s last
  documented act was to **stop treating that engine as the product**.

---

## Historical Evolution and Pivot

| When | What |
| --- | --- |
| `224b221` 2026-02-17 (`v0.1.0`) | First commit |
| `#1`–`#6` Feb–Mar 2026 | Docs, Dev Container, naming platform→framework, pyproject |
| `#8` 2026-05-05 | “Finalize Core semantic processing milestone” |
| `#9` 2026-05-15 | Clean deterministic package baseline |
| `#10` 2026-05-15 | Dead-code removal, narrower exports |
| `#11`–`#12` | README presentation |
| `#13` 2026-05-18 | Contract hardening: book-only tests, terminals, state view, runtime contract tests |
| `#14` 2026-05-19 | **Legacy / architectural exploration banner** (docs only) |

Technical arc: **build a deterministic decision-engine library →
harden contracts → explicitly retire it as TradingChassis engine
direction.** The code remains; the **product claim** does not.

U3 dead-code doc: removed unused fold/telemetry/root apply exports;
kept `RiskEngine` / step APIs as extension points; deferred some root
introspection types.

Evidence:

```text
- git log --oneline
- docs/roadmap/dead-code-cleanup-candidates.md
- CHANGELOG.md
```

---

## Evidence Index

| Area | Paths |
| --- | --- |
| Public API | `tradingchassis_core/__init__.py` |
| Pipeline | `core/domain/processing_step.py`, `processing.py` |
| Events / order | `core/domain/types.py`, `event_model.py`, `processing_order.py` |
| State / view | `core/domain/state.py` |
| Candidates | `core/domain/candidate_intent.py`, `intent_combination.py` |
| Policy | `core/domain/policy_risk_decision.py`, `core/risk/*` |
| Execution control | `core/execution_control/*`, `core/domain/execution_control_apply.py` |
| Results | `core/domain/step_result.py`, `step_decision.py` |
| Tests | `tests/semantics/` |
| Examples | `examples/core_step_quickstart.py`, `core_step_with_risk_engine.py` |
| Docs vs code | `README.md`, `docs/code-map/core-pipeline-map.md`, `docs/flows/control-time-and-scheduling.md` |
| Legacy | `README.md` banner, `7c01945` |
| CI | `scripts/check.sh`, `.github/workflows/tests.yaml` |

Web search: **not used**.

---

## Limitations and Non-Claims

Do not use this repository as evidence for:

- Building or operating the **current** TradingChassis trading engine.
- Production execution, venue connectivity, or Kubernetes trading
  Runtime (that work is **not this package**; `core-runtime` is out of
  scope here).
- Solved backtest/live **parity** or identical fills.
- Full OMS / accepted-order state machine.
- Institutional risk (the built-in `RiskEngine` is a small gate set).
- Core performing wall-clock scheduling or order dispatch.
- Property-based or replay-hash determinism proofs.
- PyPI publishing.
- Present-tense “powers TradingChassis today.”

Keep:

> Designed and implemented a deterministic event-driven trading
> decision-engine prototype exploring shared backtest/live semantics,
> intent pipelines, policy admission, and execution-control
> boundaries — later marked legacy when TradingChassis moved away from
> a custom engine.

---

## Derived Defensible Experience Statements

Valid only with the limits above.

- Architected a Pydantic-typed canonical Event boundary and reducers
  for book market data, thin order lifecycle, fills, and control time,
  with trade-shaped market events **explicitly rejected**.
- Implemented `run_core_step` and batch **wakeup** APIs that reduce
  ordered `EventStreamEntry` values, then evaluate strategy **once**
  per step/batch.
- Separated **Intents** from venue orders and combined them with
  deterministic **cancel > replace > new** dominance on
  `client_order_id`.
- Split **policy admission** (protocol + optional `RiskEngine`) from
  **Execution Control** sendability (queue, inflight, token bucket) and
  from **execution outcomes** (`OrderRejectedEvent`).
- Returned `dispatchable_intents` plus a **non-canonical** rate-limit
  `ControlSchedulingObligation`, leaving sleep/dispatch/`ControlTimeEvent`
  injection to the Runtime.
- Snapshot strategy state behind `StrategyStateView` (`MappingProxyType`
  + frozen sub-views) so evaluators do not receive queue/inflight
  reducers by default.
- Locked the public root API and pipeline semantics with pytest, mypy,
  ruff, and import-linter in CI, including a wheel reinstall check.
- Documented the project as **legacy architectural exploration** after
  TradingChassis pivoted away from implementing a custom trading
  engine.

Those statements are invalid if rewritten as “I built the production
TradingChassis engine,” “solved live/backtest parity,” or “Core
executes orders.”
