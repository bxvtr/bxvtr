# Deribit History Client

Evidence-based engineering and learning record for
[`bxvtr/deribit-history-client`](https://github.com/bxvtr/deribit-history-client).

This file is the canonical portfolio analysis record for this project. The
repository remains the technical source of truth for implementation details.
This file is not a CV, README rewrite, or marketing summary.

Repository analyzed: local clone `main` at `70c1e70` (2026-05-20).
Tag: **`v0.1.0`** (`9029bac`, 2026-02-19). Package version `0.1.0`.
License: MIT, copyright `bxvtr` (2025). Git authorship on this clone:
**all 3 commits / 7 shortlog entries by `bxvtr`**.

How to read status labels:

| Label | Meaning in this file |
| --- | --- |
| Implemented | Present in Python, schemas, scripts, tests, or workflows |
| Unit-tested | Covered by mocked pytest tests that do not hit the network |
| Externally integration-tested | Live Deribit call behind `@pytest.mark.external` |
| Optional | Off the default path (must be invoked explicitly) |
| Dependency-provided | Behavior from `requests`, `jsonschema`, or `genson` |
| Deribit-provided | Exchange API, historical host, timestamps, trade fields |
| Generated | Produced by `scripts/generate_schemas.py`, not hand-authored |
| Documented but not implemented | README / CHANGELOG / CONTRIBUTING / SECURITY claim without matching code |
| Historical / deprecated | N/A — library code did not change after the first commit |
| Planned | N/A — no TODOs in the tree |
| Inferred | Reasonable conclusion from multiple artifacts |
| Uncertain | Insufficient evidence |

Source-of-truth rule: **implementation wins over README/CHANGELOG**.

Attribution rule: **Deribit** operates the public historical market-data
API (matching, storage, exchange timestamps, rate limits, payload
shape). This repository implements a **synchronous Python HTTP client**
around a small subset of those public endpoints, plus optional
JSON-Schema artifacts and a manual live-check method. It does not
implement Deribit, a historical data platform, or a market-data
infrastructure stack.

Sources used in this record:

- **Repository evidence** — Python, tests, schemas, packaging, CI, git.
- **Deribit documentation** — official API reference (fetched 2026-08-20)
  for endpoint names, public/unauthenticated semantics, `count` maximum,
  `has_more`, and trade-field meanings. Cited only to classify provider
  vs client, not as proof of what this repo implements.
- **Local verification** — `jsonschema.validate` against the committed
  wrapper documents (see API Response-Shape Validation).
- **Web search** — used to locate Deribit docs and community mentions of
  `history.deribit.com`. Not used as proof of client behavior.

---

## Project Overview

**Status:** Implemented as a small src-layout Python package
(`deribit-history-client` 0.1.0) that wraps three Deribit **public**
GET endpoints with `requests`, returns JSON dicts, and optionally
compares live JSON-RPC envelopes to bundled Genson-generated schema
files.

What it is:

- A **thin, synchronous** client: `DeribitHistoryClient` plus a
  separate request module (`read.py`).
- Access to **instrument metadata** and **sequence-window public
  trades** via Deribit’s historical HTTP host.
- **Development tooling** to snapshot live responses into JSON Schema
  files, and a **manually invoked** `perform_api_check()` that loads
  those files and prints pass/fail.
- Mocked unit tests, an opt-in live test, Sphinx autodoc, Ruff lint CI,
  and GitHub Pages docs.

What it is not:

- Deribit, an exchange matching engine, or a historical-data generator.
- An authenticated / private-account client (no API keys, OAuth, or
  signatures).
- An HFT or market-microstructure collector (no order book, websocket,
  local receive timestamps, or latency measurement).
- A pagination engine, retry/backoff layer, or rate-limit controller.
- A monitoring service or continuous API-compatibility guarantee.
- A PyPI-published distribution (install from git / editable).
- A runtime microservice (Compose / Dev Container are **dev-only**).

Evidence:

```text
- src/deribit_history_client/client.py
- src/deribit_history_client/read.py
- src/deribit_history_client/schemas/
- scripts/generate_schemas.py
- pyproject.toml
- README.md
```

---

## What I Built

Attribution is strong for **this tree**: single-author history from
`9029bac` (library, schemas, tests, CI, docs) through two later
documentation PRs. This provides strong evidence that I was the primary
author and maintainer of the client, its tests, tooling, and project
documentation. It does **not** mean Deribit endpoints, trade timestamps,
or JSON Schema validation semantics were invented here.

### Authored in this repo — Implemented

1. **`DeribitHistoryClient` facade** — class attributes for base URL and
   headers; four methods; no instance state; no constructor arguments.
2. **Low-level GET helpers** in `read.py` — one function per endpoint,
   `requests.get(..., timeout=30)`, `response.json()`.
3. **Endpoint mapping** — Python names to Deribit public paths,
   including the non-obvious
   `get_trades_by_sequence` → `get_last_trades_by_instrument`.
4. **`raw` switch** — default returns `response["result"]`; `raw=True`
   returns the full JSON-RPC envelope.
5. **Bundled schema artifacts** — three JSON files with generation
   metadata wrapping a Genson Draft schema.
6. **`scripts/generate_schemas.py`** — live calls + Genson
   `SchemaBuilder.add_object` + overwrite of schema files (development
   tooling, not runtime learning).
7. **`perform_api_check()`** — sequential live calls, load schema via
   `importlib.resources`, print results; does not raise on
   `ValidationError`.
8. **Mocked unit tests** for the three data methods (`raw=True` only).
9. **Opt-in external test** that calls `perform_api_check()` against
   live Deribit.
10. **Packaging / docs / lint** — setuptools src layout, Sphinx autodoc,
    Ruff workflow, GitHub Pages workflow, Dev Container.

### Generated (not hand-authored schemas)

The three files under `src/deribit_history_client/schemas/` are
**generated artifacts**. Metadata records
`$generated_at: 2025-06-15T16:28:13Z` and
`$schema_generated_from` equal to the local method name. The inner
`"schema"` object is Genson output from one live response per endpoint.

### Deribit-provided / dependency-provided

- Deribit: historical HTTP API, instrument and trade payloads, exchange
  trade `timestamp`, `trade_seq`, `has_more`, JSON-RPC envelope fields
  (`jsonrpc`, `usIn`, `usOut`, `usDiff`, `testnet`).
- `requests`: TLS, HTTP GET, JSON decode, 30s timeout enforcement.
- `jsonschema`: Draft validation engine (see unwrap caveat below).
- `genson`: schema inference from sample objects.

### Documented but not implemented (current `main`)

| Claim | Where | Reality |
| --- | --- | --- |
| GitHub Actions for **testing** | README, CHANGELOG `[0.1.0]` | Workflows are lint + docs only. No pytest job. |
| `python scripts/generate_schemas.py` regenerates schemas | README, CONTRIBUTING | Script has **no** `__main__` / argparse; importing it does nothing. |
| Pre-generated schemas **included with the package** | README, CHANGELOG | Files exist in the source tree. `pyproject.toml` has no `package-data` / `MANIFEST.in`. Local `SOURCES.txt` omits the JSON files. |
| “Explicit error handling” / payload-size limits | CONTRIBUTING, SECURITY | HTTP status is not checked; no size limits; no custom exceptions. |
| CI expected to catch test failures | SECURITY | No test workflow. |

Library Python did not change after `9029bac`. Later commits only
touch README / CONTRIBUTING / SECURITY.

Evidence:

```text
- src/deribit_history_client/client.py
- src/deribit_history_client/read.py
- scripts/generate_schemas.py
- .github/workflows/lint.yaml
- .github/workflows/docs.yaml
- CHANGELOG.md
- src/deribit_history_client.egg-info/SOURCES.txt
```

---

## Public API

**Status:** Implemented. Unit-tested for the three fetch methods with
`raw=True` only. `perform_api_check` is optionally live-tested, not
unit-tested.

`src/deribit_history_client/__init__.py` is empty. Callers import:

```python
from deribit_history_client.client import DeribitHistoryClient
```

The class has no `__init__`. Shared configuration is class attributes:

| Attribute | Value |
| --- | --- |
| `BASE_URL` | `https://history.deribit.com/api/v2/public/` |
| `HEADERS` | `{"Accept": "application/json"}` |

### Methods

| Method | Parameters | Returns | Network |
| --- | --- | --- | --- |
| `get_instrument` | `instrument_name: str`, `raw: bool = False` | `Dict[str, Any]` | Yes |
| `get_instruments` | `currency`, `kind`, `expired: bool`, `raw=False` | `Dict[str, Any]` | Yes |
| `get_trades_by_sequence` | `instrument_name`, `start_seq: int`, `end_seq: int`, `count: int = 10_000`, `raw=False` | `Dict[str, Any]` | Yes |
| `perform_api_check` | defaults: `BTC`, `BTC-PERPETUAL`, `kind="future"`, `expired=False`, `start_seq=end_seq=99_999_999` | `None` | Yes (three calls) |

There is **no** generic request helper on the public class, **no**
time-range method, **no** async API, **no** pandas / pydantic models.

Return shape:

- `raw=False` (default): `response["result"]` only. Missing `"result"`
  raises `KeyError` (not handled).
- `raw=True`: full decoded JSON object.

No client-side validation of instrument names, sequence bounds,
`start_seq > end_seq`, `count`, or `kind` enums. Values are forwarded
to Deribit.

Evidence:

```text
- src/deribit_history_client/client.py
- src/deribit_history_client/__init__.py
- tests/test_client.py
```

---

## HTTP Request Layer

**Status:** Implemented. **Not** unit-tested (`read.py` is patched away
in tests, never executed).

Library: **`requests==2.32.5`** (pinned). Synchronous. Not `httpx`, not
`aiohttp`.

| Concern | Implementation |
| --- | --- |
| URL construction | `f"{base_url}/<path>"` |
| Query params | `params={...}` on GET |
| Timeout | `timeout=30` (single value: connect+read) |
| TLS | `requests` default `verify=True` (dependency-provided) |
| Status handling | **None** — no `raise_for_status()`, no `status_code` check |
| JSON decode | `response.json()`; decode errors propagate |
| Retries / backoff | **None** |
| Rate-limit / 429 / Retry-After | **None** |
| Session reuse | **None** — each call is `requests.get` (new `Session` per call in requests 2.x) |
| Custom User-Agent | **None** (requests default) |
| Auth headers | **None**. `HEADERS` is only `Accept: application/json` |
| Logging | **None** in `read.py` |

`BASE_URL` already ends with `/`, so the f-string produces a **double
slash**, e.g. `https://history.deribit.com/api/v2/public//get_instrument`.
**Inferred:** many HTTP stacks still route this; the client does not
normalize paths. Not tested in-repo.

`read.py` docstrings mention “headers including authentication
information.” That does not match `HEADERS` or any signing code.
**Documented in docstring, not implemented.**

CONTRIBUTING’s “No hidden retries, silent fallbacks, or implicit API
behavior” **is** reflected here: failures surface as raw
`requests` / `json` / `KeyError` exceptions. There is also no
explicit HTTP-error mapping.

Evidence:

```text
- src/deribit_history_client/read.py
- src/deribit_history_client/client.py (BASE_URL, HEADERS)
- pyproject.toml
- CONTRIBUTING.md
```

---

## Endpoint Abstraction

**Status:** Implemented. Three public GET paths. Mapping is in `read.py`
only — do not infer extra endpoints from README wording.

| Client method | HTTP path (appended to `BASE_URL`) | Query parameters sent |
| --- | --- | --- |
| `get_instrument` | `get_instrument` | `instrument_name` |
| `get_instruments` | `get_instruments` | `currency`, `kind`, `expired` as `"true"` / `"false"` **strings** |
| `get_trades_by_sequence` | **`get_last_trades_by_instrument`** | `instrument_name`, `start_seq`, `end_seq`, `count` |

Not sent, though Deribit’s public API documents them on the trade
endpoint (**Deribit documentation**, not this repo): `start_timestamp`,
`end_timestamp`, `sorting`. This client is sequence-window only.

API errors vs HTTP errors:

- The client does **not** inspect a JSON-RPC `"error"` object.
- Deribit public methods are documented as unauthenticated
  (**Deribit documentation**: `security: []` / Public tag).
- An HTTP 200 body with `"error"` and no `"result"` would fail later
  at `response["result"]` when `raw=False`, not as a dedicated API
  error type.

Result extraction happens only in `client.py` (`raw` flag). `read.py`
always returns the full decoded body.

Evidence:

```text
- src/deribit_history_client/read.py
- src/deribit_history_client/client.py
```

Deribit documentation (classification only):

- <https://docs.deribit.com/api-reference/market-data/public-get_instrument>
- <https://docs.deribit.com/api-reference/market-data/public-get_instruments>
- <https://docs.deribit.com/api-reference/market-data/public-get_last_trades_by_instrument>

---

## Historical Trade Retrieval

**Status:** Implemented as **one HTTP GET per call**. Not a pagination
engine.

`get_trades_by_sequence(instrument_name, start_seq, end_seq, count=10_000)`
forwards the four parameters and returns JSON. There is:

- no loop on `has_more`
- no cursor / continuation token
- no chunking of `start_seq`–`end_seq`
- no client-side sort, deduplication, or gap detection
- no special empty-result branch

`has_more` appears in the **generated schema** (`result.required`) and
in Deribit’s official response schema. The client never reads it.

### Sequence bounds

`start_seq` and `end_seq` are passed through. Inclusive/exclusive
behavior is **Deribit-provided** (docs: first/last sequence numbers
to be returned). This repo does not re-specify or test boundaries.

Default `count=10_000`. **Deribit documentation** for
`public/get_last_trades_by_instrument`: `count` default `10`, maximum
`1000`. The client does not clamp `count`. How `history.deribit.com`
treats `10000` (reject vs cap) is **Uncertain** from this repository.

`perform_api_check` uses `start_seq = end_seq = 99_999_999` — a tiny
probe, not a backfill.

The generation-script docstring example uses `start_seq=1`,
`end_seq=999_999_999` (still one request, still subject to `count`).

### Ordering / duplicates / gaps

Not implemented in the client. Any ordering is whatever Deribit
returns (`sorting` is not passed; Deribit default is “order they left
the database”).

Do not describe this as “I implemented sequence-based pagination.”
Accurate: **I mapped a sequence-window query onto one Deribit GET.**

Evidence:

```text
- src/deribit_history_client/read.py (get_trades_by_sequence)
- src/deribit_history_client/client.py (get_trades_by_sequence, perform_api_check)
- src/deribit_history_client/schemas/get_trades_by_sequence.json
```

---

## Timestamp and Data Semantics

**Status:** Client returns Deribit JSON as-is. No extra timestamps.

### What the client does **not** add

Searched `src/`, `scripts/`, `tests/`: no `datetime.now` on responses,
no receive-time fields, no latency math, no websocket sequence, no
packet timestamps. (`generate_schemas.py` stamps `$generated_at` on
**schema files**, not on trade rows.)

The README timestamp caveat is therefore **consistent with the code**
and should be kept:

- Historical trade rows contain **exchange timestamps** (Deribit’s
  `timestamp` on the trade).
- No local receive timestamp.
- No network-latency-adjusted timestamp.

### Fields present in the **generated** trade schema

From `schemas/get_trades_by_sequence.json` (one Genson sample). Required
on each trade item:

`amount`, `contracts`, `direction`, `index_price`, `instrument_name`,
`mark_price`, `price`, `tick_direction`, `timestamp`, `trade_id`,
`trade_seq`.

Optional in that schema (properties, not `required`): `combo_trade_id`,
`combo_id`.

`result` also requires `trades` (array) and `has_more` (boolean).

Envelope fields required by the generated schema: `jsonrpc`, `result`,
`testnet`, `usIn`, `usOut`, `usDiff`. Those `us*` fields are
**Deribit JSON-RPC server-side timing** on the response envelope, not
client-observed receive times, and not trade event times.

### Deribit documentation (not all of this is in the generated schema)

Official `public_trade` documents additional **optional** fields this
schema does not list, including `liquidation`, `iv`, `block_trade_id`,
`block_trade_leg_count`, `starbase_timestamp`, `starbase_match_id`,
`block_rfq_id`. `contracts` is documented as optional and “may be
absent in historical trades,” but Genson marked it **required** from
the sample that had it.

Trade `timestamp`: Deribit — milliseconds since UNIX epoch **of the
trade**. Not a client clock.

Instrument schema (generated from a **future** sample, likely
`BTC-PERPETUAL`) required-includes future-oriented fields such as
`future_type`, `max_leverage`, `max_liquidation_commission`. That is a
sample-overfit contract, not Deribit’s full instrument union (options
would add `strike` / `option_type`, etc.).

### HFT / microstructure non-claim

The tree contains no order-book snapshots, incremental book updates,
websockets, local receive times, or latency probes.

This project is **not** an HFT- or microstructure-dataset collector.
The README already states the data is unsuitable for latency-sensitive
simulations. That bound stands.

Evidence:

```text
- src/deribit_history_client/client.py
- src/deribit_history_client/read.py
- src/deribit_history_client/schemas/get_trades_by_sequence.json
- src/deribit_history_client/schemas/get_instrument.json
- README.md (Timestamp Caveat)
- scripts/generate_schemas.py ($generated_at only)
```

---

## API Response-Shape Validation

**Status:** Implemented as a **manual method** with bundled artifacts.
Intended as **response-shape** checking, not semantic compatibility.
**Effectiveness of the live check is currently limited** (see unwrap).

### Where schemas live

```text
src/deribit_history_client/schemas/get_instrument.json
src/deribit_history_client/schemas/get_instruments.json
src/deribit_history_client/schemas/get_trades_by_sequence.json
```

No `schemas/__init__.py`. On a source-tree / `PYTHONPATH=src` import,
`deribit_history_client.schemas` loads as a **namespace package**
(verified locally on Python 3.14). `perform_api_check` uses:

```python
files("deribit_history_client.schemas").joinpath(f"{name}.json")
```

Library: **`jsonschema==4.25.1`**. Trigger: only `perform_api_check()`,
not on every fetch.

### File structure

Each file is a **wrapper**, not a bare Draft document:

```json
{
  "$schema_generated_from": "<method name>",
  "$generated_at": "2025-06-15T16:28:13Z",
  "schema": { "$schema": "http://json-schema.org/schema#", "type": "object", ... }
}
```

The inner object has `type`, `properties`, `required`. It does **not**
set `additionalProperties: false` (Genson default: extra keys allowed).
Types are JSON Schema types (`integer`, `number`, `string`, `boolean`,
`array`, `object`). Nested `result` / `trades[]` / instrument objects
are present. Enums (e.g. Deribit `tick_direction` 0–3, `direction`
buy/sell) are **not** encoded — only `integer` / `string`.

### What `perform_api_check()` actually does

1. Builds a dict by **eagerly** calling all three fetch methods with
   `raw=True` (three sequential live GETs; not parallel).
2. For each name, opens `{name}.json` via package resources.
3. Missing file: print skip, `continue` (`FileNotFoundError` only).
4. `jsonschema.validate(instance=response, schema=schema)` where
   `schema` is the **entire wrapper document**.
5. Success: print `✅`. `ValidationError`: print `❌` and
   `exc.message`. **Does not re-raise.** Returns `None`.
6. Other exceptions (network, JSON decode, `ModuleNotFoundError` for
   the schemas namespace) **are not** caught.

This is **not** a monitoring service, cron, or CI gate. It is a method
a caller must invoke (or that the external pytest hits).

Observability: **`print`**, not `logging`, not metrics.

### Detectable shape drift vs semantic drift

JSON Schema, **if the inner Draft document were applied**, could flag:

- missing required keys
- type changes (string vs number, object vs array)
- nested structure changes that violate `properties` / `required`

It would **not** automatically detect:

- same types, different meaning or units
- changed sort order
- changed sequence-window semantics
- new optional fields (because `additionalProperties` is unset)
- value-range / enum tightening (`direction` stays `string`)

Prefer the phrase **API response-shape drift detection** over “detects
API changes” or “guarantees compatibility.”

### Unwrap caveat (verified)

`jsonschema` ignores unknown keywords. The wrapper’s keys
(`$schema_generated_from`, `$generated_at`, `schema`) are **not** Draft
keywords that apply the nested schema. Passing the wrapper is therefore
equivalent to validating against `{}`.

Local check with `jsonschema` (same files as the package):

| Instance | Wrapper document (what `client.py` passes) | Inner `"schema"` object |
| --- | --- | --- |
| `{"totally": "wrong"}` | **passes** | fails (`jsonrpc` required) |
| envelope with incomplete trade | **passes** | fails (`contracts` required in this generated schema) |

So the **intent** (store a contract, compare live JSON) is in the repo;
the **current `validate()` call does not apply the nested schema**.
Error reporting for shape drift via this method is therefore not
effective until the inner document is unwrapped (not implemented).

The generation script still stores a real inner schema that *could*
catch missing fields if passed correctly. That is a maintenance
artifact, not a working runtime check today.

Evidence:

```text
- src/deribit_history_client/client.py (perform_api_check)
- src/deribit_history_client/schemas/*.json
- pyproject.toml (jsonschema)
```

---

## Schema Generation and Maintenance

**Status:** Implemented as a **development function**. Not a CLI.
Not runtime auto-learning.

`scripts/generate_schemas.py`:

- Inserts the **repo root** on `sys.path` (the installable package
  actually lives under `src/`; an editable install is the realistic
  import path).
- Instantiates `DeribitHistoryClient` and calls the three methods with
  `raw=True` (**live network**).
- `genson.SchemaBuilder(); builder.add_object(response); to_schema()`
  — **one response object per endpoint**, not a corpus of many
  independent calls.
- Wraps the Genson output with `$schema_generated_from` and UTC
  `$generated_at` (seconds precision, `Z` suffix).
- Writes `src/deribit_history_client/schemas/{name}.json` relative to
  **current working directory** (must be repo root). Overwrites.
  `indent=2`. No backup, no review prompt, no diff.
- Prints `✅ Schema for {name} saved.`
- **No** `if __name__ == "__main__"`. README’s
  `python scripts/generate_schemas.py` does not invoke
  `generate_api_check_file`.

Docstring example dated **June 15, 2025** matches committed
`$generated_at`. Git first commit is **2026-02-19**: schemas were
generated earlier (or on a clock set to that date) and committed later.
**Inferred:** not regenerated in the later docs-only commits.

`genson==1.3.0` is a **runtime** install dependency even though only
the maintenance script imports it.

CONTRIBUTING asks for manual review before committing regenerated
schemas. There is no automated check that schemas match live traffic.

### Generated-schema limitations

From one live object per endpoint:

- Fields absent in that sample never appear (false sense of
  completeness).
- Fields present become Genson `required` (overfit; e.g. `contracts`).
- Heterogeneous lists (`get_instruments`) infer a **union-ish** item
  schema from items in **that one array**, still one HTTP response.
- Nullable / omitted optional Deribit fields are poorly represented.
- Output is **not** bit-stable: `$generated_at` changes every run.

Treat bundled JSON as **a snapshot of one sample shape**, not as
Deribit’s OpenAPI.

Evidence:

```text
- scripts/generate_schemas.py
- src/deribit_history_client/schemas/*.json
- CONTRIBUTING.md (Schema Updates)
- README.md (Regenerating JSON Schemas)
- pyproject.toml
```

---

## Error Handling and Operational Boundaries

**Status:** Thin pass-through. No custom exception hierarchy.

| Failure | Behavior |
| --- | --- |
| HTTP 4xx/5xx | Ignored; `.json()` may succeed or raise |
| JSON decode | Propagates (`requests` / `json`) |
| Timeout / connection | Propagates (`requests`) |
| Deribit JSON-RPC `"error"` | Not detected; `raw=False` may `KeyError` on `"result"` |
| Schema `ValidationError` | Caught **only** in `perform_api_check`; printed |
| Missing schema file | Print skip; continue |
| Invalid caller params | Forwarded to Deribit |
| Swallowed fetch failures | No — except schema validation in the check method |

### Authentication boundary

**Implemented:** public GET only. No API key, secrets, OAuth, HMAC, or
private/account endpoints. SECURITY.md states the same; the code
matches.

`.gitignore` ignores `.env`, keys, and certs. Docs deploy uses
`GITHUB_TOKEN` (GitHub-provided for Pages).

### Rate limits

None in code (no sleep, no 429 handling). SECURITY.md tells **users**
to respect Deribit policy. Deribit documents a distinct low sustained
rate for `get_instruments` (**Deribit documentation**). This client is
not rate-limit aware.

### Input validation

Only transform: `expired` bool → `"true"`/`"false"` strings. Everything
else is pass-through.

### Data transformation

None: no pandas, no typed models, no timestamp conversion, no sort,
no dedup. **Design: keep the client thin** and return dicts.

Evidence:

```text
- src/deribit_history_client/read.py
- src/deribit_history_client/client.py
- SECURITY.md
```

---

## Testing: Mocked vs External

**Status:** Two layers, correctly separated by a pytest marker. Unit
tests do not hit the network. External tests are **opt-in** and **not**
run in CI.

### Unit tests — Unit-tested / no live calls

`tests/test_client.py` uses `unittest.mock.patch` on
`deribit_history_client.read.get_instrument` /
`get_instruments` /
`get_trades_by_sequence`, returning
`DUMMY_RESPONSE = {"success": True, "result": "mocked_data"}`.

Three tests: each calls the client with `raw=True`, asserts the dummy
dict, `assert_called_once()`. Fixture `wrapper` is a bare
`DeribitHistoryClient()`.

Guarantees:

- The facade delegates to `read.*` (for those methods, `raw=True`).
- **No** `requests` I/O in the default `pytest` run.

Does **not** cover: `raw=False` / `["result"]` extraction, `read.py`
URLs/params, HTTP errors, schema validation, `perform_api_check`,
double-slash URLs, or `count` defaults.

Default pytest (`pytest.ini`): `addopts = -m "not external"`.

### External test — Externally integration-tested (opt-in)

`tests/integration/test_api_check_live.py`:

```python
@pytest.mark.external
def test_api_check_runs_against_live_api() -> None:
    client = DeribitHistoryClient()
    client.perform_api_check()
```

- Hits live Deribit (no credentials).
- **No assertions** beyond “did not raise.”
- Given the schema unwrap caveat, a silent pass does **not** prove
  shape equality.
- Excluded by default; `pytest -m external` to run.
- **Not** executed by GitHub Actions.

Determinism: unit tests use a static dummy dict (deterministic).
External tests depend on live API availability, instrument existence
(`BTC-PERPETUAL`), and network. Schema files are frozen at 2025-06-15
generation time (freshness is a maintenance concern, not a test
assertion).

Evidence:

```text
- tests/test_client.py
- tests/config.py
- tests/conftest.py
- tests/integration/test_api_check_live.py
- pytest.ini
```

---

## Packaging, Documentation and CI

### Packaging

| Item | Value |
| --- | --- |
| Name / version | `deribit-history-client` `0.1.0` |
| `requires-python` | `>=3.9,<3.12` |
| Classifier | Python **3.11** only (no 3.9/3.10 classifiers) |
| Build | setuptools ≥ 61, wheel; `package-dir = {"" = "src"}` |
| Runtime deps | `requests==2.32.5`, `jsonschema==4.25.1`, `genson==1.3.0` |
| Extra `dev` | pytest 8.3.5, ipdb, debugpy, ruff 0.8.4 |
| Extra `docs` | sphinx, sphinx-autodoc-typehints, sphinx-rtd-theme, sphinx-rtd-dark-mode |
| PyPI publish | **Not present** (README: git + editable) |

“Dependency-light” in the README is marketing tone. Concrete surface:
**three pinned runtime packages**, two of which (`jsonschema`, `genson`)
serve validation/generation rather than HTTP.

Schema package data: **not declared**. README/CHANGELOG claim bundled
schemas in the package. Source tree has the files; a local egg-info
`SOURCES.txt` lists `client.py` / `read.py` / `__init__.py` but **not**
`schemas/*.json`. After a non-editable install, `perform_api_check` may
fail to resolve the schemas namespace or skip files.
**Uncertain** for a hypothetical wheel; **not configured** in
`pyproject.toml`.

### Documentation — Implemented (Sphinx autodoc)

- `docs/source/conf.py`: autodoc, napoleon, typehints, RTD theme +
  dark mode, `release = "0.1.0"`.
- `docs/source/api.rst` autodocs `client` and `read` from **docstrings**.
- `sys.path.insert(0, abspath("../src"))` is evaluated from
  `docs/source/`, i.e. `docs/src` — **wrong relative path**. CI still
  works because it `pip install -e .[docs]`.
- Makefile `BUILDDIR = build` → `docs/build/html`. README tells readers
  to open `docs/_build/html/index.html`. CI publishes `docs/build/html`.
  `.gitignore` ignores `docs/_build/`.
- `html_static_path = ["_static"]` / `templates_path` directories are
  not in the tree.

GitHub Pages: `docs.yaml` on push to `main` + `workflow_dispatch`,
Python 3.11, `peaceiris/actions-gh-pages@v3`, `force_orphan: true`,
`permissions: contents: write`. README badge points at
`https://bxvtr.github.io/deribit-history-client/`.

### CI

| Workflow | Trigger | What it runs | Pytest? | Live API? |
| --- | --- | --- | --- | --- |
| `lint.yaml` | push `main`, PRs | `pip install ruff` (unpinned in CI) then `ruff check .` | No | No |
| `docs.yaml` | push `main`, dispatch | `pip install -e .[docs]`, sphinx-build, Pages | No | No |

No cache, no matrix beyond a single 3.11, no package-publish job.
**External marker is irrelevant to CI because pytest never runs.**

CHANGELOG `[0.1.0]`: “GitHub Actions workflows (lint, **CI**, docs)” —
the **CI/test** workflow does not exist on `main` (also absent in the
first commit). Documented but not implemented.

### Docker / Dev Container — dev environment only

- `.devcontainer/dev.Dockerfile`: `python:3.11-slim`, venv, apt includes
  `libpq-dev` and `postgresql-client` with a comment about TimescaleDB —
  **unused by this HTTP client** (likely a copied template).
- `docker-compose.yaml`: service `app`, bind-mount `.:/app`,
  `sleep infinity`. Named volume `deribit-history-client` is **declared
  and never mounted**.
- `docker-compose.override.yaml`: SELinux `:Z` bind.
- `devcontainer.json`: compose those files, `remoteUser: root`,
  `postCreateCommand` `pip install -e .[dev,docs]`.

Not a production image, not a required runtime sidecar, not a
microservice.

Evidence:

```text
- pyproject.toml
- pytest.ini
- docs/source/conf.py
- docs/source/api.rst
- docs/Makefile
- .github/workflows/lint.yaml
- .github/workflows/docs.yaml
- docker-compose.yaml
- .devcontainer/dev.Dockerfile
- .devcontainer/devcontainer.json
```

---

## Engineering Decisions and Trade-offs

1. **Thin client over a public API.** Facade + `read.py` + raw dicts.
   Easy to follow. Cost: no pagination, no HTTP error mapping, no
   models.
2. **Split transport from API methods.** `read.py` is independently
   patchable (tests use that). Cost: `read.py` itself is untested.
3. **Synchronous `requests` with a 30s timeout, no retries.** Matches
   CONTRIBUTING’s “no hidden retries.” Cost: callers must handle
   transience and 429s.
4. **Sequence window, not time window.** Maps onto Deribit’s
   `start_seq` / `end_seq`. Cost: no
   `get_last_trades_by_instrument_and_time` helper; `count` default
   disagrees with Deribit’s documented maximum.
5. **Store generated JSON Schema as versioned artifacts; compare live
   JSON later.** This is a real **contract-as-artifact** pattern. Cost:
   Genson-from-one-sample overfitting; wrapper not unwrapped at
   validate time; check is print-only and optional.
6. **Marker-gated live test vs mocked units.** Correct isolation.
   Cost: live test has no assertions; CI never runs pytest at all.
7. **Pin runtime deps; CI installs latest Ruff.** Reproducible app
   install; lint version can drift from `dev` extra.
8. **Public historical host hardcoded.** No constructor override for
   `BASE_URL` / testnet. Simple. Cost: awkward to point at
   `test.deribit.com` without patching the class attribute.
9. **Dev Container copied with Postgres packages.** Convenient if the
   same template is used next to a Timescale helper. Cost: noise and a
   unused volume in Compose.
10. **Keep `genson` on the runtime extra-less dependency list.** One
    `pip install` gets everything. Cost: generation library in
    production installs.

---

## Key Technical Learnings

Only where this tree supports them:

- **A public-API client can stay small** if it refuses retries,
  pagination, and dataframe conversion — and that refusal must be
  described as a bound, not as “resilience.”
- **Request helpers vs facade** makes mocking easy; it also hides
  untested URL/param code unless `read.py` is tested directly.
- **JSON Schema is not “API compatibility.”** It is (at best)
  **response-shape** checking. Semantic drift (units, meaning, ordering)
  will not show up. Extra fields will not show up without
  `additionalProperties: false`.
- **Generated schemas overfit the sample.** Required `contracts` vs
  Deribit’s “may be absent historically” is the concrete example.
- **Metadata wrappers must be unwrapped** before `jsonschema.validate`.
  Storing `$generated_at` beside the Draft document is useful; passing
  the wrapper as the schema is equivalent to `{}`.
- **Exchange timestamps ≠ client receive timestamps.** Returning
  Deribit JSON unchanged is the correct honesty for historical REST.
  It also means this dataset cannot support latency or microstructure
  claims.
- **`has_more` in the payload is not pagination** unless the client
  loops. Documented provider fields are easy to mistake for implemented
  client behavior.
- **Mocked tests and an opt-in live smoke test serve different purposes.**
  In this repository the separation is explicit, but the live test has no
  assertions and CI never runs pytest, so neither layer currently acts as a
  GitHub-hosted regression gate.
- **Packaging generated JSON** needs `package-data` / `MANIFEST.in`.
  Files in `src/.../schemas/` are not automatically a wheel artifact.
- **Keeping private/authenticated endpoints out of scope** is a
  security and product decision visible in headers, SECURITY.md, and
  the three public paths.

---

## Historical Evolution

Three commits; **library code is frozen** at the first:

| When | What |
| --- | --- |
| `9029bac` 2026-02-19 (`v0.1.0`) | Client, `read.py`, three Genson schemas, generate script, mocked tests, external marker, Sphinx, lint + docs workflows, Compose/Dev Container, README/CHANGELOG/SECURITY |
| `58a342d` 2026-04-14 | README presentation (`#1`) |
| `70c1e70` 2026-05-20 | README / CONTRIBUTING / SECURITY wording (`#2`) |

Technical arc: **snapshot a small public-history client with a
schema-check idea in v0.1.0**, then documentation only. Schema files
were not regenerated in git after the snapshot (`$generated_at` remains
2025-06-15). No endpoint-mapping fixes appear in history because there
were no follow-up code commits.

Git used: `log`, `ls-tree` of `9029bac`, `diff --stat 9029bac HEAD`,
`shortlog`, tag `v0.1.0`.

---

## Evidence Index

| Area | Paths |
| --- | --- |
| Facade | `src/deribit_history_client/client.py` |
| HTTP | `src/deribit_history_client/read.py` |
| Schemas | `src/deribit_history_client/schemas/*.json` |
| Generation | `scripts/generate_schemas.py` |
| Unit tests | `tests/test_client.py`, `tests/config.py`, `tests/conftest.py` |
| Live test | `tests/integration/test_api_check_live.py` |
| Markers | `pytest.ini` |
| Package | `pyproject.toml` |
| CI | `.github/workflows/lint.yaml`, `docs.yaml` |
| Docs | `docs/source/conf.py`, `api.rst`, `index.rst` |
| Dev env | `.devcontainer/*`, `docker-compose.yaml` |
| Claims vs code | `README.md`, `CHANGELOG.md`, `CONTRIBUTING.md`, `SECURITY.md` |

Web search: **used** for Deribit official API reference (endpoint
semantics, `count` max, public/unauthenticated, trade field catalog)
and to confirm community use of `history.deribit.com`. Official OpenAPI
`servers` examples use `https://test.deribit.com/api/v2`; this client’s
host is hardcoded to `history.deribit.com` (**repository evidence**).

---

## Limitations and Non-Claims

Do not use this repository as evidence for:

- Building Deribit, a matching engine, or “historical market data
  infrastructure.”
- HFT-grade data, microstructure research, or latency-sensitive
  replay (no books, no websockets, no receive timestamps).
- Client-observed or network-adjusted timestamps.
- A pagination / backfill engine (single GET; `has_more` ignored).
- Retries, backoff, connection pooling, or rate-limit handling.
- Authenticated trading, account history, or private `historical=true`
  user-trade APIs (those are a different Deribit feature set).
- Guaranteed API compatibility or a monitoring product.
- Effective live JSON Schema enforcement **as currently wired**
  (wrapper document passed to `validate`).
- Hand-authored, review-complete OpenAPI-equivalent schemas.
- Pytest on GitHub Actions.
- PyPI publishing.
- A production microservice or required Docker runtime.
- “Secure exchange integration” beyond “public GET, verify TLS
  defaults, no secrets in the client.”

Keep the sentence:

> Implemented a Python client around Deribit’s public historical API
> with endpoint abstractions and development tooling for schema-based
> response-shape checks.

Keep the qualification explicit: schema artifacts and a live-check method
exist, but the current `jsonschema.validate` call does not apply the nested
Draft schema, so effective shape enforcement is not implemented yet.

---

## Derived Defensible Experience Statements

Valid only with the limits above.

- Designed a small src-layout Python package that wraps Deribit’s
  public historical REST API (`history.deribit.com`) with a facade
  (`DeribitHistoryClient`) and a separate `requests` GET layer.
- Mapped three public endpoints (`get_instrument`, `get_instruments`,
  `get_last_trades_by_instrument`) including the Python name
  `get_trades_by_sequence` versus the real HTTP path.
- Returned decoded JSON dicts with an explicit `raw` flag rather than
  introducing pandas or ORM models.
- Kept the client unauthenticated and timeout-bounded (`timeout=30`)
  without hidden retries, matching a documented “no implicit transport
  magic” rule.
- Added development tooling that snapshots live JSON-RPC responses
  with Genson and commits the result as versioned schema artifacts
  (`$schema_generated_from`, `$generated_at`).
- Wired an optional `perform_api_check()` that performs live calls and
  is intended to compare responses to those artifacts — with the
  documented unwrap limitation on `jsonschema.validate`.
- Split **mocked unit tests** (patch `read.*`, default pytest excludes
  live) from an **opt-in** `@pytest.mark.external` smoke test against
  Deribit.
- Packaged the project for Python `>=3.9,<3.12` with pinned runtime
  dependencies, Ruff, Sphinx autodoc, and GitHub Pages — CI running
  **lint and docs**, not pytest.

Those statements are invalid if expanded to “I built a market-data
platform,” “HFT historical capture,” “rate-limit-aware resilient
client,” “pagination engine,” “CI-tested live Deribit suite,” or
“JSON Schema guarantees API compatibility.”
