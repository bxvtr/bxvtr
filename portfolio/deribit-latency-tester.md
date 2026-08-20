# Deribit Latency Tester

Evidence-based engineering and learning record for
[`bxvtr/deribit-latency-tester`](https://github.com/bxvtr/deribit-latency-tester).

This file is the canonical portfolio analysis record for this project. The
repository remains the technical source of truth for implementation details.
This file is not a CV, README rewrite, or marketing summary.

Repository analyzed: local clone `main` at `9e2d114` (2026-05-20).
Tag: **`v0.1.0`** (`cccb3ba`, 2026-02-19). Crate version `0.1.0`, edition
2021. License: MIT, copyright `bxvtr` (2025). Git authorship on this
clone: **all 6 commits / 11 shortlog entries by `bxvtr`**.

How to read status labels:

| Label | Meaning in this file |
| --- | --- |
| Implemented | Present in Rust source, config, or workflows |
| Live-capable | Can talk to a live Deribit WebSocket if credentials exist |
| Testnet-capable | `testnet = true` → `wss://test.deribit.com/ws/api/v2` |
| Mainnet-capable | `testnet = false` → `wss://www.deribit.com/ws/api/v2` (committed default) |
| Mainnet-validated | Evidence of an actual mainnet run — **not present** |
| Client-observed | Timestamp taken on this process (`Instant` / `Utc`) |
| Exchange-provided | Field originated at Deribit (`usIn` / `usOut` / `usDiff`) |
| Derived | Computed locally from other timestamps |
| Configured | Taken from `config.toml` or environment |
| Tested | Covered by `#[test]` — **none in this tree** |
| Untested | No automated test |
| Documented but not implemented | README / CHANGELOG / CONTRIBUTING claim without matching code |
| Historical | N/A for library logic — `src/` did not change after first commit |
| Inferred | Reasonable conclusion from multiple artifacts |
| Uncertain | Insufficient evidence |

Source-of-truth rule: **implementation wins over README**.

Attribution rule: **Deribit** operates matching, WebSocket JSON-RPC,
authentication, and the `usIn` / `usOut` / `usDiff` fields. This
repository implements an **async Rust measurement client** that places
**real limit orders**, records application-observed round-trip times,
stores exchange-reported processing fields, and optionally correlates
those RPCs with application-level raw-book receive times. It does not
implement Deribit, a matching engine, colocation, or a packet-capture
latency stack.

Sources used in this record:

- **Repository evidence** — Rust, `config.toml`, CSV writer, CI, git.
- **Deribit documentation** (web search, 2026-08-20) — JSON-RPC
  extensions, `public/auth`, `post_only` / `reject_post_only`. Used only
  to classify provider semantics, not as proof of local behavior.
- **Engineering inference** — labeled as such (for example: session
  binding after discarded tokens; Ping auto-reply with a split stream).

---

## Project Overview

**Status:** Implemented, Live-capable, Testnet-capable, **Mainnet-capable**
(committed `config.toml` has `testnet = false`). Untested as a suite.
Not Mainnet-validated from this repository.

What it is:

- A **binary crate** (`src/main.rs`, no `lib.rs`) that connects over
  **TLS WebSocket**, authenticates with **client credentials**, and
  runs `num_iterations` of **open → edit → cancel** private order RPCs.
- A **measurement path**: monotonic and wall-clock timestamps around
  each timed RPC, plus Deribit `usIn` / `usOut` / `usDiff` copied into
  CSV.
- Optional **raw-book subscription** used only to stamp the most recent
  locally received `book.*` frame (orders are **not** tick-triggered).
- CSV logging with per-row flush and a floor-rank percentile summary.

What it is not:

- A network / NIC / kernel / colocation benchmark.
- An execution-latency or fill-to-ack benchmark (fills are ignored).
- A tick-to-trade trigger or causal market-reaction study.
- A harmless dry-run: it sends **`private/buy` or `private/sell`**.
- A reconnecting trading daemon, risk engine, or cancel-on-disconnect
  watchdog.
- A PyPI/crates.io published library (no `[lib]`, no publish workflow).

Evidence:

```text
- src/main.rs
- src/deribit_client.rs
- src/latency.rs
- src/summary.rs
- src/config.rs
- config.toml
- Cargo.toml
```

---

## What I Built

Attribution is strong for **this tree**: `cccb3ba` contains the full
Rust implementation; later commits are Compose/Dev Container and
README/docs. This provides strong evidence that I was the primary author
and maintainer of the measurement client and its project tooling. It does
**not** mean Deribit timestamps, matching, or TLS were invented here.

### Authored in this repo — Implemented

1. **Tokio binary** with a split WebSocket, reader task, oneshot RPC
   correlation, and a sequential order loop on the main task.
2. **Config overlay** — TOML for non-secrets; `DERIBIT_CLIENT_ID` /
   `DERIBIT_CLIENT_SECRET` from the environment.
3. **Testnet/mainnet URL branch** with no further safety interlock.
4. **`public/auth`** (`grant_type: client_credentials`) immediately
   after connect.
5. **Timed order lifecycle** — `private/buy` or `private/sell` (limit,
   `post_only: true`), `private/edit`, `private/cancel`.
6. **Price path** — ticker `mark_price`/`last_price` (else
   `base_price`), percent offset, `f64` nearest-tick quantization.
7. **Latency logger** — CSV schema, engine-field extraction, error
   extraction, ack-to-ack delta, flush-per-row.
8. **Summary printer** — min / floor-rank median / p90 / p99 / max.
9. **Optional raw-book subscribe** and last-tick `RwLock`.
10. **Dev Container + Compose** (idle `sleep infinity`) and a CI job
    (fmt, clippy `-D warnings`, empty `cargo test`).

### Deribit-provided / dependency-provided

- Deribit: WebSocket JSON-RPC, auth, private order methods, book
  channel, `usIn` / `usOut` / `usDiff`.
- `tokio` / `tokio-tungstenite` / `rustls`: runtime, TLS WS, split
  sink/stream.
- `chrono`: UTC wall clock.
- `csv` / `serde` / `toml` / `anyhow`: IO and errors.

### Documented but not implemented

| Claim | Where | Reality |
| --- | --- | --- |
| Sleep “to avoid rate limits” as adaptive control | README | Fixed `tokio::time::sleep` only; no 429 handling |
| “Full Docker” runs the tester | README | Compose command is `sleep infinity` |
| Nanosecond **accuracy** | README “nanosecond precision” | `as_nanos()` **representation**; RTT stored as **microseconds** |
| CLI-free implying no clap | README | `clap` is an **unused** Cargo dependency |
| CONTRIBUTING “deterministic / reproducible timing” | CONTRIBUTING | Live exchange, no timeouts, wall clock subject to NTP |

`thiserror` is also unused in `src/`.

Evidence:

```text
- src/main.rs
- src/deribit_client.rs
- Cargo.toml
- README.md
- CONTRIBUTING.md
- docker-compose.yaml
```

---

## Runtime Architecture

**Status:** Implemented. Untested.

Startup (`#[tokio::main] main`):

1. `Config::load_from_file("config.toml")` (cwd, not argv).
2. `program_start = Instant::now()` — epoch for all `*_mono_ns`.
3. Unbounded `mpsc` for `MarketDataEvent`.
4. `DeribitClient::connect` — TLS WS + `public/auth`.
5. Spawn MD last-tick writer.
6. Optional `public/subscribe` `book.{instrument}.raw`.
7. `public/get_instrument` → `tick_size` (default `0.5` if missing).
8. `public/ticker` → `mark_price` else `last_price`; on error use
   `cfg.base_price`. **Not refreshed per iteration.**
9. `LatencyLogger::new` (`File::create` **truncates**).
10. `run_roundtrip_test` — sequential timed RPCs.
11. Optional `print_summary_from_csv`.
12. Print `Done.` and exit. **No WebSocket Close, no cancel-all, no
    Drop cleanup.**

### Tasks and ownership

| Task | Role |
| --- | --- |
| Main | Connect, subscribe, ticker, timed RPCs, CSV, summary |
| Reader (`tokio::spawn` in `connect`) | Owns split **stream**; stamps Text; routes RPC vs subscription |
| MD tracker (`tokio::spawn` in `main`) | Overwrites `last_tick_ns` for `book.*` channels |

Shared state:

| Object | Type | Purpose |
| --- | --- | --- |
| `pending` | `Arc<Mutex<HashMap<i64, oneshot::Sender<RpcResponse>>>>` | JSON-RPC id → waiter |
| `next_id` | `Arc<Mutex<i64>>` | Starts at `1`, increment after assign |
| `ws_tx` | split `Sink` on **main** (`&mut self` `send_rpc`) | Serialized sends |
| `last_tick_ns` | `Arc<RwLock<Option<i64>>>` | ns since `program_start` of last book Text |
| `order_id_state` | `Arc<Mutex<Option<String>>>` | Last open `order_id` (single-threaded user) |

**How send and receive avoid blocking each other:** the WebSocket is
**split**. The reader loop never holds `ws_tx`. `send_rpc` writes the
sink then `await`s a oneshot filled by the reader. Only one timed RPC
is in flight at a time because `send_rpc` takes `&mut self` and the
order loop is sequential. The MD task can update `last_tick_ns`
concurrently via `RwLock`.

There is **no** cooperative cancellation, Ctrl-C handler, or reader
join. If the reader task ends, leftover oneshots are **not** failed
explicitly.

Evidence:

```text
- src/main.rs
- src/deribit_client.rs
```

---

## WebSocket and JSON-RPC Model

**Status:** Implemented. Live-capable. Untested.

### Connection

| Item | Implementation |
| --- | --- |
| Testnet URL | `wss://test.deribit.com/ws/api/v2` |
| Mainnet URL | `wss://www.deribit.com/ws/api/v2` |
| TLS | `tokio-tungstenite` `0.21` + `rustls-tls-native-roots` |
| Connect | `connect_async` — **no timeout** |
| Reconnect | **None** (README matches code) |
| Client Close | **None** |
| Heartbeat RPC | Ignored (“can be ignored for now”) |
| Application Ping | Received and **not** answered in this code |

Comment in `deribit_client.rs` claims tungstenite “usually” auto-pongs.
With a **split** stream this is **Uncertain**. Do not claim heartbeat
robustness.

On `Message::Close` or stream `Err`, the reader **breaks**. Subsequent
`ws_tx.send` may error (program stops). If send already succeeded, the
oneshot waiter **can hang forever** — `rx.await` has no timeout.

### JSON-RPC correlation — Implemented (untested)

Request shape:

```json
{"jsonrpc":"2.0","id":<i64>,"method":<method>,"params":<params>}
```

`send_rpc`:

1. Lock `next_id`, take id, increment, drop lock.
2. Create oneshot; **insert into `pending` before send** (avoids
   “response before waiter” on this socket).
3. `serde_json::to_string` → `ws_tx.send(Message::Text)`.
4. `rx.await` until the reader `remove`s that id and `send`s.

Reader routing:

- Top-level `id` as `i64` → RPC path. String ids would **miss**.
- Else `method == "subscription"` → `MarketDataEvent` (channel name
  only; book payload unused).
- Else ignore (heartbeats, etc.).
- JSON parse failure: Text was already timestamped, then **dropped**;
  waiter still blocked.

Late / unknown id: silently dropped. Duplicate ids: not possible with
the monotonic counter unless `i64` wraps (not handled). Dropped
oneshot receiver: `let _ = tx.send(resp)`.

Auth, subscribe, `get_instrument`, and ticker use `send_rpc` **without**
`timed_rpc` → **not in the CSV**.

Evidence:

```text
- src/deribit_client.rs
```

---

## Authentication and Trading Boundary

**Status:** Implemented. Live-capable. Credentials: Configured via env.

### Credentials

- Read from `DERIBIT_CLIENT_ID` and `DERIBIT_CLIENT_SECRET`.
- Empty strings rejected.
- **Not** in `config.toml`.
- `.gitignore` covers `.env` / `.env.*`. No committed `.env.example`
  despite `!.env.example`.
- **No `dotenv` crate.** README’s `source .env` is a **shell** step;
  the binary only sees the process environment.

### Auth RPC

`public/auth` with `grant_type: "client_credentials"` and **raw
`client_secret` in JSON params** (expected for this grant; sent on
WSS).

`access_token` / `refresh_token` / `expires_in` in the result are
**discarded**. Later private methods do **not** attach `access_token`.
The design assumes the **WebSocket session stays authenticated** after
the auth RPC.

**Deribit documentation:** private methods normally take `access_token`
unless the connection was authenticated with a **session** scope so the
server remembers the token. This code does not set `scope`. Whether
Deribit binds a default WS session is **exchange-provided** and
**Uncertain** without a live run. If it does not bind, private RPCs
would error and still be logged as error rows.

No token refresh, no expiry handling, no re-auth.

### Logging / redaction

- Credentials are not written to CSV.
- Request JSON is not `println!`’d.
- Auth **failure** uses `{:?}` on `resp.error` (typically code/message,
  not the secret).
- CSV **does** store `order_id` on edit/cancel rows (local file). README
  is correct that **committed** `output/local_latency.csv` is synthetic
  and that real order ids are not meant to be committed.

### Testnet vs mainnet

`config.toml` committed default: **`testnet = false`** (mainnet URL).
There is **no** compile-time, runtime, or prompt guard. Same order RPCs
run on both hosts. **Mainnet-capable ≠ Mainnet-validated.**

### Financial risk (code-backed)

This tool can create **real orders** with whatever API key permissions
the operator configured. `order_amount`, `side`, offsets, and
`testnet` are operator-controlled. Market movement can produce **fills**.
Treat it as a **live trading client with a measurement sidecar**, not as
a read-only probe.

Evidence:

```text
- src/config.rs
- src/deribit_client.rs (authenticate, URLs)
- config.toml
- .gitignore
- output/local_latency.csv
```

---

## Order Lifecycle

**Status:** Implemented. Live-capable. Untested.

Per iteration (`0..num_iterations`):

| Phase | `op_type` | Method | Params | Timed CSV |
| --- | --- | --- | --- | --- |
| Open | `buy` or `sell` | `private/buy` or `private/sell` | `instrument_name`, `amount`, `type: "limit"`, `price`, `post_only: true` | Yes |
| Sleep | | | `sleep_between_requests` | |
| Edit | `edit` | `private/edit` | `order_id`, `amount`, `price` — **no `post_only`** | Yes, if `order_id` |
| Sleep | | | | |
| Cancel | `cancel` | `private/cancel` | `order_id` | Yes, if `order_id` |
| Sleep | | | after iteration | |

Not sent: `reduce_only`, `reject_post_only`, `time_in_force` (Deribit
default GTC), `access_token`. **Market orders are not used** (`type`
is always `"limit"`).

`order_id` is taken from `result.order.order_id` **after** the open
sample is already written, so **open CSV rows have empty `order_id`**
(matches the committed synthetic CSV). If missing, edit/cancel are
skipped (`No active order_id to edit/cancel.`). The open error/success
row is still logged.

**Fills:** **not inspected.** No `order_state`, `filled_amount`,
`trades`, or user-trade subscription. A filled or cancelled-by-exchange
order still proceeds to edit/cancel; those RPCs are timed and logged,
including error bodies.

Evidence:

```text
- src/main.rs (run_roundtrip_test, timed_rpc)
```

---

## Price Construction and Order Safety

**Status:** Implemented formulas. Safety is **partial**, not a fill
guarantee.

### Reference price

```text
base = ticker.mark_price OR ticker.last_price
     OR cfg.base_price on any ticker error
```

Bid/ask are **not** used. The value is **fixed for the run**.

### Open (buy and sell share one formula)

```text
open_raw = base * (1.0 + price_offset_percent / 100.0)
open     = quantize_price(open_raw, tick_size)
```

Committed defaults: `price_offset_percent = 5.0`, `side = "sell"`.
A **sell 5% above** mark is typically away from the market. A **buy
with the same +5%** is **through** the market, not below it.

README text about BUY moving below the market describes the **edit
sign**, not the open formula. **Code wins.**

### Edit

```text
Buy:  edit_offset = price_offset_percent - edit_offset_step_percent
Sell: edit_offset = price_offset_percent + edit_offset_step_percent
new   = quantize(base * (1.0 + edit_offset / 100.0), tick_size)
```

Defaults: sell edit **5.5% above**; buy edit **4.5% above** (still
above if open used +5%). Negative `edit_offset_step_percent` reverses
further-vs-closer, as README states.

### Tick size / amount

```rust
fn quantize_price(price: f64, tick_size: f64) -> f64 {
    if tick_size <= 0.0 { return price; }
    (price / tick_size).round() * tick_size
}
```

Nearest tick, **f64**, not Decimal, not buy-floor/sell-ceil. Off-grid
prices remain possible. Missing `tick_size` → **`0.5`**. **Amount is
the config `f64` as JSON** — no `min_trade_amount`, no contract-size
conversion, no rounding. Units are whatever Deribit expects for that
instrument (operator responsibility).

### `post_only` is not “cannot fill”

Open sets `post_only: true` and does **not** set `reject_post_only`.

**Deribit documentation:** with `post_only` and default
`reject_post_only: false`, a crossing order is **repriced to one tick
inside the spread**, not rejected. It can **rest at the touch and
fill**. Reject-if-would-take requires `reject_post_only: true` (not
implemented). Edit omits `post_only` entirely.

Fallback `base_price = 10000` with instrument `BTC_USDC-PERPETUAL`: if
the ticker call fails, a **sell** at ~10500 would be far through a
~spot BTC market. `post_only` may slide that order to the touch rather
than abort.

No cancel-on-error, cancel-on-disconnect, or Ctrl-C cancel-all.

**Do not claim “safe non-executing orders.”** Default sell+5% is a
**mitigation**, not a lockout.

Evidence:

```text
- src/main.rs (quantize_price, fetch_ticker_price, open/edit params)
- config.toml
```

Deribit documentation (classification only):
<https://docs.deribit.com/api-reference/trading/private-buy>

---

## Measurement Model

Four **clock domains** are used. They are **not inter-calibrated**.

| Domain | Source | Stored as | Comparable to |
| --- | --- | --- | --- |
| A. Client monotonic | `std::time::Instant` | ns since `program_start`; RTT in **µs** | Other `Instant`s in this process |
| B. Client wall | `chrono::Utc` | RFC3339; `rtt_wall_us` | Other UTC instants (NTP can step) |
| C. Exchange processing | JSON `usIn`/`usOut`/`usDiff` | i64 as received | Each other (Deribit Unix **µs**) |
| D. Book arrival | `Instant` on WS Text | `tick_ts_mono_ns` | Domain A only |

`Instant` can **represent** nanoseconds (`as_nanos()`). That is **not**
nanosecond measurement **accuracy** (clock resolution, scheduling,
syscall jitter). RTT is truncated with `as_micros()`. Prefer:
**recorded in nanosecond units on a monotonic clock; reported RTT in
microseconds.**

Auth/subscribe/instrument/ticker RPCs are **outside** the sample set.

Evidence:

```text
- src/main.rs (program_start, timed_rpc)
- src/deribit_client.rs (recv timestamps)
- src/latency.rs
```

---

## Monotonic RTT

**Name used here:** **application-observed WebSocket RPC round-trip
time** (`rtt_mono_us`).

| Bound | When | Where |
| --- | --- | --- |
| **Start** | `Instant::now()` in `timed_rpc` **before** `send_rpc` | `src/main.rs` |
| **End** | `Instant::now()` on `Message::Text` **before** `serde_json::from_str` | `src/deribit_client.rs` |
| **Formula** | `(recv_ts_mono - send_ts_mono).as_micros() as i64` | `latency.rs` `duration_us` |

**Includes (client-observed path):** `next_id` mutex, oneshot +
`pending` insert, JSON serialize, sink send, TLS write, network,
Deribit queues + handling, TLS read, tungstenite delivering Text,
then Instant.

**Excludes:** JSON parse, pending lookup, oneshot wake, return to
`timed_rpc`, CSV serialize/flush.

**Not:** NIC hardware timestamp, kernel packet timestamp, pure network
RTT, matching-engine queue time alone, or fill latency.

Because start is **before** serialize/locks/write, local queueing is
**inside** the number. Because end is **before** parse, client JSON
decode is **outside**. Still **application-level**, not wire RTT.

### Wall-clock RTT — Implemented (derived)

README “RTT (mono + wallclock)” is true for **CSV columns**:

```text
send_ts_wall = Utc::now()     // immediately before send Instant
recv_ts_wall = Utc::now()     // immediately after recv Instant
rtt_wall_us  = recv.signed_duration_since(send).num_microseconds().unwrap_or(0)
```

Wall RTT is a **second clock**, useful for logs / joining to other UTC
series, **not** a better latency meter (NTP steps). Summary statistics
use **`rtt_mono_us` only**.

Evidence:

```text
- src/main.rs (timed_rpc)
- src/deribit_client.rs (Text handler)
- src/latency.rs (duration_us, duration_wall_us)
- src/summary.rs
```

---

## Exchange-Reported Processing

**Status:** Implemented extraction. Exchange-provided values.
**Not** combined with RTT in code.

| Field | CSV column | Deribit documentation (2026-08-20) |
| --- | --- | --- |
| `usIn` | `engine_us_in` | UTC µs when the **request was received** |
| `usOut` | `engine_us_out` | UTC µs when the **response was sent** |
| `usDiff` | `engine_us_diff` | Server-side handling µs, documented as `usOut - usIn` |

Extracted from the **top-level JSON-RPC object** (`resp.raw`), as i64
or truncated f64. Missing keys → `None`. **Not** converted, **not**
aligned to `program_start` or `Utc`, **not** applied to notifications
(subscriptions have no sample row). Error responses may still carry
these fields (**Deribit**); the client will store them if present.

**`RTT − usDiff` is not computed.** Deribit docs suggest comparing
total RTT with `usDiff` to discuss network delay. Even if it were
computed, the residual would still include client serialize/scheduling,
non-engine server work, and WS overhead — call it **RTT residual after
subtracting exchange-reported processing**, never “network latency.”
This repo does not form that residual.

Evidence:

```text
- src/latency.rs (extract_engine_timestamps)
```

Deribit documentation (classification only):
<https://docs.deribit.com/articles/json-rpc-overview>

---

## Raw-Book and Tick-Relative Timing

**Status:** Optional (`subscribe_raw_book`). Client-observed receive
times. **Not causal.**

Subscribe RPC: `public/subscribe` with
`channels: ["book.{instrument}.raw"]`. Error is printed; success is
printed if `error` is absent. Channel payload (sequence, bids/asks,
exchange book timestamp) is **discarded**.

**Stamp time:** `Instant::now()` on the same Text frame as RPC
responses — **application-level receive timestamp** after tungstenite
delivers the frame, **not** after JSON parse of the book body (parse
happens next; the Instant is already taken). Not a NIC/pcap stamp, not
Deribit’s book `timestamp` field.

MD task: if `channel.starts_with("book.")`, overwrite
`last_tick_ns = (recv_ts_mono - program_start).as_nanos()`.

`timed_rpc` **snapshots** `last_tick_ns` **before** send Instant. A
book update during the in-flight RPC does **not** change that row.

Derived (same monotonic domain), ns→µs via `/ 1000.0` then `.round()`:

```text
tick_to_send_us = round((send_ts_mono_ns - tick_ts_mono_ns) / 1000)
tick_to_ack_us  = round((recv_ts_mono_ns  - tick_ts_mono_ns) / 1000)
```

If no book frame yet: both `None`.

README is correct that the book **does not gate** order send. Orders
run on a fixed loop + sleep. Therefore:

- These are **tick-relative timing metrics** (age of the last locally
  received book frame at send / at ack).
- They are **not** tick-to-trade, market-reaction, or “the exchange
  processed this book state.”
- They are **not** proximity-to-matching-engine proof.

`ack_delta_prev_us` is **not** book-related: rounded µs between this
sample’s recv Instant and the previous sample’s recv Instant (includes
configured sleeps). First row `None`.

Evidence:

```text
- src/main.rs (subscribe, MD task, timed_rpc snapshot)
- src/deribit_client.rs (subscription branch)
- src/latency.rs (tick_to_* formulas)
- README.md (does not determine when orders are sent)
```

---

## CSV Evidence and Summary Statistics

**Status:** Implemented. Untested. Local files may contain real order
ids; the **committed** sample is synthetic (README + implausible
placeholder `USDC-12345678901`).

### Schema (`RoundtripSample`)

| Column | Origin |
| --- | --- |
| `op_type` | `buy` / `sell` / `edit` / `cancel` |
| `rpc_method` | e.g. `private/sell` |
| `instrument_name` | config |
| `order_id` | argument to `timed_rpc` (empty on open) |
| `tick_ts_mono_ns` | last book recv, ns since start |
| `send_ts_mono_ns` / `recv_ts_mono_ns` | Instant vs `program_start` |
| `send_ts_wall_iso` / `recv_ts_wall_iso` | RFC3339 UTC |
| `rtt_mono_us` / `rtt_wall_us` | derived |
| `tick_to_send_us` / `tick_to_ack_us` | derived or empty |
| `engine_us_in` / `engine_us_out` / `engine_us_diff` | exchange JSON |
| `error_code` / `error_msg` | `error.code` / `error.message` |
| `ack_delta_prev_us` | derived |

No credentials columns.

### Durability

`File::create` truncates each run. `serialize` + **`flush` every
row**. A crash loses at most the in-flight RPC not yet flushed (and
any OS buffers after flush — usual caveat). Parent dirs are created.

RPC **body** errors: still one sample row, then the loop continues
(README matches). Transport errors: `send_rpc` `Err` → `timed_rpc`
fails → **process stops**.

### Summary

Re-reads the CSV. Series: all `rtt_mono_us`; optional tick/ack-delta
vectors. Per series: `sort_unstable`, min, max, `percentile` at 50/90/99.

```text
rank = (p / 100.0) * (n - 1)
idx  = floor(rank)   // no interpolation, no mean
```

Small `n` (default **3 samples** per iteration): p90/p99 often equal
median. Engine columns unused. README example `count: 100` is
**illustrative**, not from the committed CSV (3 rows).

Evidence:

```text
- src/latency.rs
- src/summary.rs
- output/local_latency.csv
- README.md
```

---

## Failure Handling and Safety Boundaries

| Event | Behavior |
| --- | --- |
| Auth error | Process exits |
| Subscribe error | Print; continue without ticks |
| Ticker error | `base_price` fallback (**risk**) |
| `get_instrument` error | Process exits |
| Timed RPC transport error | Process exits |
| Timed RPC JSON-RPC `error` | Log row; continue |
| Missing `order_id` | Skip edit/cancel; open row already logged; **no** extra cancel-all |
| Reader death after send | **`rx.await` can hang** (no timeout) — stronger than README’s “send_rpc returns an error” |
| Disconnect with live order | **No cleanup** — GTC order can remain |
| Ctrl-C | **No** handler — same dangling-order risk |
| 429 / rate limit | **No** handling; only fixed sleep |
| Reconnect | **None** |
| Explicit RPC/connect timeout | **None** |

README “no reconnect” and “no explicit timeout” match. README “socket
error → program stops” is true for **send** failure, incomplete for
**silent reader exit**.

Evidence:

```text
- src/deribit_client.rs
- src/main.rs
- README.md (Error Handling)
```

---

## Configuration and Secret Handling

**Status:** serde TOML + env. Weak validation.

`FileConfig` fields are all required (missing key = parse error). No
defaults in code. No checks for: empty instrument, `num_iterations == 0`
(succeeds, no samples), negative amount, absurd offsets, mainnet
confirmation, path traversal. `Duration::from_secs_f64` **panics** on
negative / NaN / inf.

Hardcoded config path `"config.toml"` in cwd. **No CLI** (despite
unused `clap`).

Evidence:

```text
- src/config.rs
- config.toml
- src/main.rs (load_from_file("config.toml"))
```

---

## Testing and CI

**Status:** CI Implemented. Tests: **none** (`#[test]` / `tests/`
absent). Measurement functions, percentiles, price math, CSV, and
correlation are **Untested**.

`.github/workflows/ci.yaml`:

- push/PR to `main`
- `permissions: contents: read`
- `ubuntu-latest`, `dtolnay/rust-toolchain@stable`, `Swatinem/rust-cache@v2`
- `cargo fmt --all -- --check`
- `cargo clippy --all-targets --all-features -- -D warnings`
- `cargo test --all-features` (empty suite → pass)

No Deribit credentials, no live job, no `cargo audit`, no release
artifact. **CI does not place orders** and does **not** prove
measurement correctness.

`Cargo.lock` is **gitignored** (comment even says keep it for
binaries). CI resolves `tokio = "1"` etc. at build time — **not** a
pinned lockfile build.

**Live validation:** no git/docs evidence of a real testnet or mainnet
run. Committed CSV is labeled synthetic. **Live-capable ≠ live-tested.**

Dev Container: `rust:1.83`, clippy/rustfmt, Compose `sleep infinity`.
Not a production image.

Evidence:

```text
- .github/workflows/ci.yaml
- .gitignore (/Cargo.lock)
- .devcontainer/dev.Dockerfile
- docker-compose.yaml
```

---

## Engineering Decisions and Trade-offs

1. **Split WS + oneshot map** so the receive loop can stamp Text and
   complete waiters without holding the sink. Cost: reader death can
   hang `send_rpc`; Ping not answered in-app.
2. **Timestamp before `send_rpc` / on Text before parse.** Simple and
   close to “message appeared.” Cost: RTT includes local serialize and
   locks; excludes parse.
3. **Keep monotonic RTT and wall RTT as separate columns.** Honest
   about clock domains. Cost: wall RTT is easy to misuse as “the”
   latency.
4. **Store Deribit `usIn`/`usOut`/`usDiff` raw; do not subtract from
   RTT.** Avoids a false “network latency” column. Cost: no residual
   metric at all.
5. **Tick timestamps as analytics only, not a trigger.** README and
   code agree. Cost: easy to over-read as tick-to-trade.
6. **Real private limit-order RPCs instead of a public-only ping.**
   This exercises the authenticated order path. Cost: **financial and
   operational risk**, especially with the mainnet default and
   `post_only` without `reject_post_only`.
7. **Fixed sleep between RPCs.** Simple pacing. Cost: not 429-aware.
8. **No reconnect / no timeout / no Ctrl-C cancel.** Matches
   “no hidden retry.” Cost: hang, dangling GTC orders.
9. **Env-only secrets, TOML for the rest.** Sensible split. Cost: no
   dotenv loader; secret still appears in the auth JSON on the wire.
10. **CI fmt+clippy, no measurement tests, lockfile ignored.** Catches
    style; not latency math or live safety.

---

## Key Technical Learnings

Only where this tree supports them:

- **JSON-RPC over a multiplexed WebSocket needs correlation** (id →
  oneshot) plus a dedicated reader; notifications (`subscription`)
  must not look like RPC replies.
- **The measurement boundary is part of the metric.** Start-before-lock
  vs start-after-write changes whether local queueing is “latency.”
- **Monotonic clocks measure intervals; wall clocks label events;
  exchange `usDiff` is a third domain.** Mixing them without saying so
  is a correctness bug in naming.
- **`as_nanos()` is representation, not accuracy.**
- **Tick-relative ≠ causal.** A book frame that arrived earlier is not
  the cause of an independent order loop.
- **`post_only` without `reject_post_only` is not “will not trade”**
  (Deribit may slide to the touch).
- **Live-order benchmarks need disconnect/cancel story.** This one
  exits without `cancel_all`.
- **Fixed sleep is pacing, not rate-limit handling.**
- **Floor-rank percentiles on n=3 are descriptive, not robust
  statistics.**
- **CI that never hits Deribit cannot validate a trading measurement
  tool** — only that it formats and clippy-cleans.

---

## Historical Evolution

`src/`, `Cargo.toml`, and `config.toml` are **unchanged** after the
first commit. Later work is environment and documentation.

| When | What |
| --- | --- |
| `cccb3ba` 2026-02-19 (`v0.1.0`) | Full Rust tester, config (mainnet default), synthetic CSV, CI, docs |
| `7bb65d0` / `f2e588b` 2026-02-19 | Compose override `:Z` mount; Dev Container compose file list |
| `c24fe2b` / `d324d6c` 2026-04-14, `9e2d114` 2026-05-20 | README / CONTRIBUTING / SECURITY wording |

Technical arc: **one-shot implementation of an order-RPC stopwatch**,
then docs. No later safety fix for `reject_post_only`, timeouts, or
mainnet default.

Git used: `log`, `diff --stat cccb3ba HEAD`, tag `v0.1.0`, `shortlog`.

---

## Evidence Index

| Area | Paths |
| --- | --- |
| Entry / lifecycle | `src/main.rs` |
| WS / RPC / auth | `src/deribit_client.rs` |
| CSV / clocks / engine fields | `src/latency.rs` |
| Percentiles | `src/summary.rs` |
| Config / secrets | `src/config.rs`, `config.toml` |
| Deps | `Cargo.toml` |
| CI | `.github/workflows/ci.yaml` |
| Sample output | `output/local_latency.csv` |
| Claims vs code | `README.md`, `CHANGELOG.md`, `CONTRIBUTING.md`, `SECURITY.md` |
| Dev env | `.devcontainer/*`, `docker-compose.yaml` |

Web search: **used** for Deribit JSON-RPC `usIn`/`usOut`/`usDiff`,
`public/auth`, and `post_only` / `reject_post_only`. Client behavior
remains repository evidence.

---

## Limitations and Non-Claims

Do not use this repository as evidence for:

- Nanosecond-**accurate** or HFT-grade latency.
- Pure **network** RTT, wire latency, kernel/NIC/colocation latency.
- **Matching-engine** or **execution** latency (fills ignored;
  `usDiff` is server handling, not “match time” proven here).
- **Tick-to-trade** or market-reaction latency.
- Adaptive rate-limit / reconnect / timeout design.
- Fill-safe or “non-executing” quoting (`post_only` slides; no
  `reject_post_only`; no disconnect cancel).
- Testnet-only operation (mainnet is the committed default).
- Mainnet- or testnet-**validated** runs (no live proof in git).
- Measurement-unit tests or lockfile-reproducible CI.
- A production trading system or risk gateway.

Keep the sentence:

> Implemented an async Rust WebSocket tool that records
> application-observed JSON-RPC round-trip times using a monotonic
> clock and stores Deribit-reported `usIn`/`usOut`/`usDiff` alongside
> each sample while exercising real private limit-order RPCs.

---

## Derived Defensible Experience Statements

Valid only with the limits above.

- Built a Tokio WebSocket client (`tokio-tungstenite` + rustls) that
  splits sink/stream, correlates JSON-RPC replies via monotonically assigned
  `i64` request ids and oneshot waiters, and demuxes `subscription`
  notifications.
- Authenticated with Deribit `public/auth` `client_credentials` from
  environment variables (secret on the WSS auth payload; tokens not
  persisted).
- Drove an `open → edit → cancel` private order loop (`private/buy` or
  `private/sell` limit + `post_only`, then `private/edit` /
  `private/cancel`) with config-driven side, amount, and percent
  offsets.
- Defined a monotonic RTT from `Instant` immediately before `send_rpc`
  to `Instant` on the inbound Text frame before JSON parse, stored as
  microseconds, with a parallel UTC wall-clock RTT column.
- Extracted exchange-provided `usIn`/`usOut`/`usDiff` into CSV without
  subtracting them from RTT.
- Subscribed to `book.<instrument>.raw` and computed **tick-relative**
  send/ack delays from application-level book-frame arrival times,
  without using the book as an order trigger.
- Wrote flushed CSV samples (including RPC error code/message) and a
  floor-rank min/median/p90/p99/max summary over `rtt_mono_us`.
- Documented (in this record) that the tool is **mainnet-capable**,
  places **real orders**, and does not implement reconnect, RPC
  timeouts, or cancel-on-disconnect.

Those statements are invalid if expanded to “I measured Deribit network
latency,” “HFT-accurate nanosecond capture,” “tick-to-trade,” “safe
paper trading,” or “CI-validated live benchmarks.”
