# Timescale Access

Evidence-based engineering and learning record for
[`bxvtr/timescale-access`](https://github.com/bxvtr/timescale-access).

This file is the canonical portfolio analysis record for this project. The
repository remains the technical source of truth for implementation details.
This file is not a CV, README rewrite, or marketing summary.

Repository analyzed: local clone `main` at `7e60eca` (2026-05-20).
Tag: **`v0.1.0`** (`aa92d82`, 2026-02-18). Package version `0.1.0`.
License: MIT, copyright `bxvtr`. Git authorship on this clone: **all 5
commits / 11 shortlog entries by `bxvtr`**.

How to read status labels:

| Label | Meaning in this file |
| --- | --- |
| Implemented | Present in Python, tests, or Compose |
| Database-provided | PostgreSQL / TimescaleDB feature the wrapper *calls* |
| Dependency-provided | pandas / SQLAlchemy doing the heavy lift |
| Integration-tested | Exercised against a real TimescaleDB container (local Compose) |
| Unit-tested only | N/A — the suite is integration-style, but incomplete |
| Historical / removed | Existed in git, deleted from `main` |
| Documented but not implemented | README/CHANGELOG claim without current files |
| Inferred | Reasonable conclusion from multiple artifacts |
| Uncertain | Insufficient evidence |

Source-of-truth rule: **implementation wins over README/CHANGELOG**.
This project is a **Python library wrapping TimescaleDB/PostgreSQL**, not
a time-series database, not a bulk-ingest engine, and not a published
runtime service.

---

## Project Overview

**Status:** Implemented as a small SQLAlchemy 2 + pandas facade
(`TimescaleAccess`) over a containerized TimescaleDB for **trade-like**
time series (`instrument_name`, `trade_seq`, `timestamp`).

What it is:

- A **src-layout** installable package (`pip install -e .`) targeting
  Python `>=3.9,<3.12` (classifier lists **3.11** only).
- Helpers to **create schemas**, **infer a simple column map from a
  DataFrame**, **append rows**, call Timescale **`create_hypertable`**,
  and run **data-quality SQL** (sequence gaps, key duplicates, null
  summaries, chunk size).
- A **Dev Container + Compose** loop that starts
  `timescale/timescaledb:2.19.3-pg14` so pytest can hit a real database.

What it is not:

- A TimescaleDB implementation, chunk planner, or compression/retention
  product.
- A generic SQL abstraction (APIs assume Deribit-style trade columns).
- A PyPI-published distribution (install from git / editable).
- A long-running service (no root runtime `Dockerfile` on `main`).
- A high-throughput ingest path with benchmarks.

Evidence:

```text
- pyproject.toml
- src/timescale_access/client.py
- docker-compose.yaml
- README.md
```

---

## What I Built

Attribution is strong for **this tree**: single-author history from
`26c03ad` (2026-02-18) through the later documentation changes. This
provides strong evidence that I was the primary author and maintainer of
the wrapper, its tests, and project documentation. It does **not** mean
PostgreSQL, Timescale hypertables, pandas `to_sql`, or SQLAlchemy
`insert().on_conflict_do_nothing()` were invented here.

### Authored in this repo — Implemented

1. **`TimescaleAccess` facade** — one class, URL in, `Engine` held for
   the object lifetime, methods delegating to `engine` / `write` /
   `read` / `analysis`.
2. **Schema ensure** — `CREATE SCHEMA IF NOT EXISTS` (identifier
   interpolated).
3. **Hypertable orchestration** — if table missing, `CREATE TABLE` from
   DataFrame dtypes; optional `ALTER TABLE ... ADD COLUMN`; pandas
   `to_sql(..., if_exists="append", method="multi")`; then
   `create_hypertable(..., migrate_data => TRUE, if_not_exists => TRUE)`
   if not already a hypertable.
4. **Conflict insert** — unique index on
   `(instrument_name, trade_seq, time_column)` then **row-by-row**
   `ON CONFLICT DO NOTHING` (not `DO UPDATE`).
5. **Read helpers** — inspect tables/columns/indexes; filtered
   `SELECT * ... ORDER BY trade_seq`; catalog queries (`pg_database`,
   `pg_roles`, `pg_stat_activity`); distinct timestamps.
6. **Analysis SQL** — `generate_series` gap fill, `LAG` non-consecutive
   seq, duplicate groups, generated PL/pgSQL null-count function,
   chunk size via `_timescaledb_catalog` + `pg_total_relation_size`.
7. **Integration tests** — 14 tests against Compose Timescale (hostname
   `db` by default).
8. **Dev Container** — Python 3.11-slim app container, Timescale
   sidecar, `pip install -e .[dev,docs]`.
9. **Sphinx autodoc** + GitHub Pages workflow.
10. **Ruff** lint workflow (not pytest).

### Database- / dependency-provided

- Timescale: `create_hypertable`, hypertable catalog, chunk files.
- PostgreSQL: `ON CONFLICT`, unique indexes, `information_schema`,
  `generate_series`, window `LAG`, `pg_size_pretty`.
- pandas: dtype inspection, `to_sql`, `read_sql`, datetime conversion
  in **tests**.
- SQLAlchemy: `Engine`, `text()`, `inspect()`, PostgreSQL
  `insert().on_conflict_do_nothing()`.

### Documented but not implemented (current `main`)

- GHCR runtime image / `docker pull ghcr.io/bxvtr/timescale-access:latest`
  — **no root `Dockerfile`**, no publish workflow.
- “GitHub Actions CI pipeline with TimescaleDB service” (CHANGELOG) —
  **removed** in `aa92d82` (see Historical Evolution).
- “High-performance” / “upserts” as stated in CHANGELOG — see ingestion
  section.

```text
Evidence:
- git log --author=bxvtr
- src/timescale_access/*.py
- tests/test_client.py
```

---

## Public API and Abstraction

**Status:** Implemented. Package root `__init__.py` is **empty**; callers
import `from timescale_access.client import TimescaleAccess`.

Constructor: `TimescaleAccess(db_url: str)` → `create_engine(db_url)`
with **no pool kwargs** (SQLAlchemy defaults only — do not claim custom
pooling).

Public methods (facade): connection check/dispose; `insert_hypertable` /
`insert_hypertable_on_conflict`; `drop_table` / `ensure_schema_exists`;
reads (`get_table`, names, distinct, indexes, timestamps); cluster
introspection (databases, roles, memberships, connections, schemas);
analysis (`get_missing_trade_seq`, `get_nonconsecutive_trade_seq`,
`get_duplicate_rows`, `get_null_summary`, `get_hypertable_size`,
`get_row_count`).

Return types: `None` for writes; `pd.DataFrame` for table/analysis;
`List`/`Dict` for catalog; `str` for pretty size; `int` for row count;
`bool` for `check_connection`.

**Design facts (not slogans):**

- One broad facade class over four modules (~1.3k LOC), with multiple
  database concerns exposed through a single public object.
- **Layer leak:** `insert_hypertable_on_conflict` lives in `analysis.py`
  though CONTRIBUTING asks for read/write/analysis separation.
- **Domain coupling:** conflict index, `ORDER BY trade_seq`, gap SQL, and
  null-summary grouping all assume `instrument_name` + `trade_seq`.
  Not a generic hypertable client.
- Default **chunksize mismatch:** client `insert_hypertable(...,
  chunksize=500)` vs `write.insert_hypertable(..., chunksize=1000)`.
  The facade always passes the argument, so class users get 500.
- Docstring in `__init__` mentions a ``connection`` module that does
  not exist (`engine.py` is used).

```text
Evidence:
- src/timescale_access/client.py
- src/timescale_access/__init__.py
- CONTRIBUTING.md
```

---

## Connection and Transaction Model

**Status:** Implemented as a long-lived Engine + per-call
`connect()` / `begin()`.

- `get_engine`: `create_engine(db_url)` only.
- `check_connection`: `SELECT 1`; **swallows all exceptions** → `False`.
- `dispose_connection`: `engine.dispose()`.
- Writes that need atomic DDL use `engine.begin()` (commit/rollback
  via context manager).
- `ensure_schema_exists` uses `connect()` + explicit `commit()` (not
  `begin()`).
- **`insert_hypertable` is not one transaction:**
  1. `begin()`: existence check, `CREATE TABLE` / `ALTER COLUMN`
  2. **`df.to_sql` on the Engine** (its own connection/transaction)
  3. `begin()`: `create_hypertable` if needed  

  If step 2 fails after step 1, a **plain table** can remain. If step 3
  fails after step 2, **rows exist** without hypertable conversion until
  a later call. No retries.

- Conflict path: first `begin()` for DDL + unique index; second
  `begin()` for row inserts + hypertable. Empty DataFrame returns
  early (`logging.info`).

No custom exceptions besides wrapping `to_sql` failures in
`RuntimeError`. SQLAlchemy errors otherwise propagate.

```text
Evidence:
- src/timescale_access/engine.py
- src/timescale_access/write.py (insert_hypertable)
- src/timescale_access/analysis.py (insert_hypertable_on_conflict)
```

---

## Schema and Hypertable Management

**Status:** Implemented wrapper around PostgreSQL DDL + Timescale
`create_hypertable`.

### Schema / tables

- `ensure_schema_exists`: `CREATE SCHEMA IF NOT EXISTS {schema_name}`
  — **identifier not quoted/validated**.
- `drop_table`: `DROP TABLE IF EXISTS {schema}.{table} CASCADE`.
- Existence: parameterized `information_schema.tables` /
  `timescaledb_information.hypertables`.

### Type inference (heuristic)

| pandas dtype | SQL type |
| --- | --- |
| integer | `BIGINT` |
| float | `NUMERIC` |
| datetime64 (any) | `TIMESTAMP` (**not** `TIMESTAMPTZ`) |
| everything else (bool, object, string, …) | `TEXT` |

No explicit timezone handling in the library. Tests convert ISO strings
and **unix ms** with `pd.to_datetime` **before** insert. Naive
`TIMESTAMP` plus SQLAlchemy `dtype={time_column: TIMESTAMP()}` on
`to_sql` only.

Unsupported dtypes are not rejected; they fall through to `TEXT`.

### Hypertable conversion

Timescale-provided `create_hypertable(regclass, time_column,
migrate_data => TRUE, if_not_exists => TRUE)`. The wrapper decides
**when** to call it (after first insert path). No `chunk_time_interval`,
space partitioning, compression, or retention APIs.

Idempotency: `if_not_exists` plus a catalog check. Adding columns on
existing tables is implemented; **type changes of existing columns are
not**.

**Unique index and hypertables:** conflict insert creates
`(instrument_name, trade_seq, {time_column})`. Including the time
column is required for unique indexes on Timescale hypertables
(database constraint). That is a **correct wrapper choice**, not a
Timescale feature invented here.

```text
Evidence:
- src/timescale_access/write.py
- tests/test_client.py (datetime conversion before insert)
```

---

## DataFrame Ingestion and Conflict Handling

**Status:** Implemented. **Not** a custom COPY/executemany engine.
**Not** upsert.

### `insert_hypertable`

Orchestration around **pandas `DataFrame.to_sql`**:

- `if_exists="append"`
- `method="multi"`
- `chunksize` from caller
- `index` default `False`

“Fast” in README is **unbenchmarked**. Describe the technique, not
throughput.

Raises `ValueError` if `time_column` missing from the DataFrame.

### `insert_hypertable_on_conflict`

1. Same CREATE/ALTER heuristics as write.
2. `CREATE UNIQUE INDEX unique_{table}_instrument_seq_ts ON ...
   (instrument_name, trade_seq, {time_column})` inside a `DO $$`
   block if the index name is absent.
3. `Table(..., autoload_with=engine)` then **one `INSERT ... ON
   CONFLICT DO NOTHING` per row** (`df.to_dict(orient="records")`).
4. Then `create_hypertable` in the same second transaction.

**`DO NOTHING` skips conflicting keys; it does not update.** CHANGELOG
“conflict-safe upserts” overstates the SQL.

Conflict columns are **hardcoded** (plus configurable time column
name). Missing those columns will fail at index/insert time, not via a
dedicated validation error.

Index-name / schema binds mix SQLAlchemy `:index_name` in the
`IF NOT EXISTS` query with **f-string interpolation** in the
`EXECUTE 'CREATE UNIQUE INDEX ...'`.

```text
Evidence:
- src/timescale_access/write.py (to_sql)
- src/timescale_access/analysis.py (insert_hypertable_on_conflict)
```

---

## Time-Series Analysis Helpers

**Status:** Implemented in SQL, returned as DataFrames. **Not covered
by tests.**

| Method | Technique | Returns |
| --- | --- | --- |
| `get_missing_trade_seq` | Per `instrument_name`, `generate_series(min,max)` minus actual `trade_seq` | All **missing IDs in the closed range** (can be large) |
| `get_nonconsecutive_trade_seq` | `LAG(trade_seq) PARTITION BY instrument_name` | Rows where `trade_seq - previous_seq != 1` (first row of a partition dropped because `LAG` is NULL) |
| `get_duplicate_rows` | `GROUP BY instrument_name, trade_seq HAVING COUNT(*) > 1` then `SELECT *` | Full rows sharing that **two-column** key (not the three-column unique index) |
| `get_null_summary` | Creates PL/pgSQL function `check_nulls_in_{schema}_{table}` if missing; loops columns; `COUNT(*)` of NULLs **grouped by `instrument_name`** | DataFrame of `(instrument_name, column_name, null_count)` |
| `get_hypertable_size` | Lookup `id` in `_timescaledb_catalog.hypertable`; `pg_size_pretty(SUM(pg_total_relation_size))` on `_timescaledb_internal` chunks `_hyper_{id}_%_chunk` | Pretty string; `ValueError` if not a hypertable. **Does not call** Timescale `hypertable_size()` |
| `get_row_count` | `SELECT COUNT(*)` | int |

Null-summary function is **not** recreated if the table later gains
columns (`EXISTS` on `pg_proc.proname` only). Dynamic SQL inside the
function uses `format(..., %L, %I)` for **column** names (safer) while
schema/table in the function body are **interpolated at CREATE time**.

No compression/retention/chunk-interval helpers exist — do not claim
that Timescale ops experience from this repo.

```text
Evidence:
- src/timescale_access/analysis.py
- tests/test_client.py  (no analysis assertions)
```

---

## SQL Construction and Safety Boundaries

**Status:** Mixed. Values sometimes bound; identifiers usually
interpolated. **Do not claim “SQL injection safe.”**

| Pattern | Where |
| --- | --- |
| Bound parameters (`:schema_name`) | table existence, hypertable catalog, `get_schemas` `NOT IN :ignored`, function-name exists check |
| Double-quoted identifiers | `get_distinct_values` only |
| f-string identifiers (schema, table, column) | CREATE/ALTER/DROP, `create_hypertable` regclass, analysis queries, `get_row_count`, `get_existing_timestamps`, most of `get_table` |
| Filter **values** interpolated | `get_table`: `BETWEEN {low} AND {high}`, `IN ('a','b')`, `= 'value'` — **not bound** |
| Integer from catalog into `LIKE` | hypertable chunk pattern `_hyper_{id}_%%_chunk` |

`SECURITY.md` lists SQL execution safety as in-scope and states the
project does **not** manage authn/authz. Compose uses hardcoded
`superuser` / `supersecret` for **local dev**. `python-dotenv` is a
pinned dependency and **unused** in `src/`.

```text
Evidence:
- src/timescale_access/read.py (get_table, get_distinct_values)
- src/timescale_access/write.py
- SECURITY.md
- docker-compose.yaml
```

---

## Testing Against TimescaleDB

**Status:** **Integration-tested locally** against Compose Timescale.
**Not run in GitHub Actions on current `main`.**

- `tests/config.py`: `DATABASE_URL` env or
  `postgresql://superuser:supersecret@db:5432/postgres` (Compose
  service name `db` — intended **inside** the app container).
- Session-scoped fixture: one `TimescaleAccess`, `dispose` in
  `finally`. **No schema teardown**; later tests depend on
  `test_insert_and_read` having created `TEST_TABLE_1/2`. **Order-
  dependent**, not parallel-safe.
- Image: `timescale/timescaledb:2.19.3-pg14`. Persistent volume
  `timescale-data` (tests can see leftover state across runs).
- No Compose **healthcheck** on `db` (`depends_on` without condition).
- 14 tests: connection, schemas, insert (ISO + unix-ms **preconverted**),
  conflict insert (existence + timestamps, **not** that duplicates were
  skipped), columns, distinct+NULL skip, indexes list type, `get_table`
  equality filter, drop, databases/roles/connections.
- **Untested:** analysis helpers, range/`IN` filters, missing
  `time_column`, hypertable catalog assertion, schema-change ALTER,
  `DO NOTHING` behavior, timezone.

Comment in `tests/config.py` still says “In CI, this is typically
provided via `DATABASE_URL`” — leftover from the **deleted** CI
workflow.

```text
Evidence:
- tests/test_client.py
- tests/conftest.py
- tests/config.py
- docker-compose.yaml
```

---

## Development Environment and Reproducibility

**Status:** Implemented for clone → Dev Container → pytest **if** the
Timescale sidecar is up.

- `.devcontainer/dev.Dockerfile`: `python:3.11-slim`, build deps +
  `libpq-dev` + `postgresql-client`, venv `/opt/venv`, **does not COPY
  the app** (bind-mount).
- `devcontainer.json`: compose `app` service, `postCreateCommand`
  editable `[dev,docs]`, pytest args `tests`.
- `docker-compose.override.yaml`: SELinux `:Z` mount.
- Reproducibility: **pinned** Timescale image tag, pinned Python deps
  in `pyproject.toml`. Compose DB credentials are **dev defaults**, not
  secrets management. Persistent DB volume reduces test isolation.

This is developer/test infrastructure, not a production deployment.

```text
Evidence:
- .devcontainer/devcontainer.json
- .devcontainer/dev.Dockerfile
- docker-compose.yaml
- docker-compose.override.yaml
```

---

## Packaging, Docker and CI

**Status:** Library packaging implemented. Runtime GHCR **not**
implemented on `main`. CI = lint + docs.

### Package

- setuptools, `package-dir = {"" = "src"}`.
- Runtime pins: `sqlalchemy==2.0.40`, `pandas==2.2.3`,
  `python-dotenv==1.1.0` (unused), `psycopg2-binary==2.9.10`.
- Optional `dev` (pytest 8.3.5, ruff 0.8.4, debug tools), `docs`
  (Sphinx RTD).
- No `[project.scripts]` entry point; not a CLI.
- Install documented as editable / `pip install git+https://...`.
  **No PyPI publish workflow.**

### Docker

Only `dev.Dockerfile`. CHANGELOG/README “GHCR-ready runtime image” is
**stale**. Historical `ci.yaml` (first commit) referenced `./Dockerfile`
and `docker/build-push-action` — that job was **deleted** in PR #1
before a runtime Dockerfile existed.

### GitHub Actions (current)

| Workflow | Trigger | What |
| --- | --- | --- |
| `lint.yaml` | push `main`, PR | `pip install ruff` (unpinned vs `ruff==0.8.4`) then `ruff check .` |
| `docs.yaml` | push `main`, dispatch | `pip install -e .[docs]`; `sphinx-build`; `peaceiris/actions-gh-pages@v3` to `gh-pages`, `contents: write` |

Actions pinned by **tag** (`checkout@v4`, `setup-python@v5`), not SHA.
No pytest job. No Timescale service.

Sphinx: autodoc of all five modules (`docs/source/api.rst`).
`conf.py` references `_templates` / `_static` directories that are
absent.

Ruff: E,F,B,I; ignore E501; exclude `docs` and `.devcontainer`.
**No mypy, black, pre-commit, coverage gate.**

```text
Evidence:
- pyproject.toml
- .github/workflows/lint.yaml
- .github/workflows/docs.yaml
- git show 26c03ad:.github/workflows/ci.yaml
- git show aa92d82  (delete ci.yaml)
```

---

## Engineering Decisions and Trade-offs

1. **Facade over SQLAlchemy instead of a custom driver.** Fast to
   expose hypertables to pandas users. Cost: identifier quoting,
   pooling, and bulk I/O stay at defaults.
2. **Heuristic dtype map + `to_sql` for the happy path.** Convenient
   for notebooks. Cost: bool→TEXT, naive TIMESTAMP, no schema
   migrations.
3. **`ON CONFLICT DO NOTHING` + unique index including time.** Correct
   for “don’t insert duplicate trades,” and satisfies Timescale unique-
   index rules. Cost: not upsert; row-by-row inserts; conflict helper
   filed under `analysis`.
4. **Analysis in PostgreSQL** (`generate_series`, `LAG`) rather than
   pandas. This keeps analysis close to the stored data. Cost: these helpers
   are untested; identifiers are interpolated; `generate_series(min,max)` can
   expand to a very large result set.
5. **Generate a per-table PL/pgSQL function for nulls.** Dynamic column
   set. Cost: stale function if schema changes; extra catalog objects.
6. **Compose + Dev Container for a real Timescale.** Honest integration
   tests. Cost: tests skip on host without `db` DNS; no CI job after
   workflow removal.
7. **Delete broken GHCR/pytest workflow rather than fix it.** `#1`
   unblocked `main`. Cost: CHANGELOG still advertises CI+GHCR.
8. **Pin runtime libraries, not CI ruff.** Reproducible installs; lint
   version can drift.
9. **Session-scoped DB fixture.** Faster. Cost: order coupling and
   leftover hypertables.
10. **No chunk interval / retention API.** Keeps v0.1.0 small. Cost:
    little “Timescale ops” surface.

---

## Key Technical Learnings

Project-specific:

- **Wrapping `create_hypertable` is orchestration, not implementing
  hypertables.** The interesting code is *when* to CREATE vs ALTER vs
  convert, and that this is **not** one transaction with `to_sql`.
- **DataFrame dtypes are a leaky schema language.** Integer/float/
  datetime/TEXT is enough for a demo ingest; bool and tz-aware data
  need an explicit map.
- **Identifier vs value parameterization is easy to get half-right.**
  Catalog lookups bind schema/table names; DDL and `get_table` filters
  often do not.
- **`ON CONFLICT DO NOTHING` is skip-on-duplicate, not upsert.** Unique
  indexes on hypertables must include the time column.
- **Gap detection has two APIs:** expand every missing id
  (`generate_series`) vs report jumps (`LAG`). They answer different
  operational questions and have different cost.
- **A real Timescale in Compose is stronger than mocks** for
  `create_hypertable`, and still weaker than CI if the workflow is
  deleted.
- **Docs can outlive deleted pipelines.** README/CHANGELOG GHCR and
  “CI with TimescaleDB service” are a maintenance lesson.
- **Domain columns in a “generic” client** (`trade_seq`) couple the
  library to one ingest shape.

---

## Historical Evolution

| When | What |
| --- | --- |
| `26c03ad` 2026-02-18 | Library, tests, Compose, Dev Container, Sphinx, lint/docs workflows, **and** a `ci.yaml` with Timescale service + pytest + GHCR push of a **missing** `Dockerfile` |
| `aa92d82` / tag `v0.1.0` | **Delete** `ci.yaml` (“Fix/workflows”); README badges adjusted. Library code unchanged |
| `f8bbb8d` | Changelog wording (still lists CI + GHCR Dockerfile) |
| `e27e06c`, `7e60eca` | README/docs presentation |

Technical arc: **intended** GitHub-hosted pytest-against-Timescale and
image publish; **current** local Compose tests + lint/docs CI only.
No API renames in the five commits; later work is documentation.

---

## Evidence Index

| Area | Paths |
| --- | --- |
| Facade | `src/timescale_access/client.py` |
| Engine | `src/timescale_access/engine.py` |
| Ingest / DDL | `src/timescale_access/write.py` |
| Conflict + analysis | `src/timescale_access/analysis.py` |
| Reads | `src/timescale_access/read.py` |
| Tests | `tests/test_client.py`, `tests/conftest.py`, `tests/config.py` |
| Compose | `docker-compose.yaml` |
| Dev env | `.devcontainer/*` |
| Package | `pyproject.toml` |
| CI now | `.github/workflows/lint.yaml`, `docs.yaml` |
| CI then | `git show 26c03ad:.github/workflows/ci.yaml` |
| Docs overclaim | `README.md`, `CHANGELOG.md` |

Git used: `log`, tags, `show` of deleted `ci.yaml`, first-commit file
list. Web search: **not used**.

---

## Limitations and Non-Claims

Do not use this repository as evidence for:

- Building TimescaleDB, PostgreSQL, or a time-series database engine.
- High-throughput or benchmarked ingestion / “production-grade data
  platform.”
- Upsert (`DO UPDATE`) — only `DO NOTHING`.
- Custom connection pooling.
- SQL injection resistance for identifiers or `get_table` filters.
- TIMESTAMPTZ / timezone-correct storage in the library itself.
- Compression, retention, continuous aggregates, or chunk-interval
  tuning.
- PyPI publishing.
- A GHCR **runtime** image or current pytest-on-Actions job.
- CONTRIBUTING’s “production-ready interface” as a verified bar.
- Analysis-helper correctness (untested).

Keep the sentence:

> Implemented a Python wrapper around TimescaleDB/PostgreSQL for
> DataFrame ingest, hypertable setup, and trade-sequence data-quality
> queries, with local integration tests against a pinned Timescale
> container.

---

## Derived Defensible Experience Statements

Valid only with the limits above.

- Designed a SQLAlchemy 2 facade (`TimescaleAccess`) that holds an
  Engine and delegates ingest, catalog reads, and analysis SQL.
- Implemented DataFrame→PostgreSQL column inference (BIGINT / NUMERIC /
  TIMESTAMP / TEXT) and automatic `CREATE TABLE` / `ADD COLUMN` before
  pandas `to_sql(..., method="multi")`.
- Orchestrated Timescale `create_hypertable(..., if_not_exists => TRUE,
  migrate_data => TRUE)` after ingest, including catalog checks via
  `timescaledb_information.hypertables`.
- Added PostgreSQL `ON CONFLICT DO NOTHING` inserts keyed by
  `(instrument_name, trade_seq, time_column)` after ensuring a unique
  index (not upsert).
- Wrote SQL-side data-quality queries: `generate_series` hole fill,
  `LAG` gap detection, duplicate groups, and a generated null-count
  function — **untested** in the suite.
- Added pytest integration tests that talk to
  `timescale/timescaledb:2.19.3-pg14` via Docker Compose / Dev
  Container (ISO and unix-ms timestamps converted in the test, not in
  the library).
- Packaged the project as an editable setuptools src layout with pinned
  runtime deps, Ruff, and Sphinx autodoc deployed to GitHub Pages.
- Removed a broken Actions workflow that assumed a runtime Dockerfile
  and GHCR push — current CI is Ruff + docs only.

Those statements are invalid if expanded to “I built Timescale,”
“upsert,” “fast ingest,” or “CI tests against Timescale on every PR.”
