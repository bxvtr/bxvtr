# TradingChassis Documentation

Evidence-based engineering and learning record for
[`TradingChassis/docs`](https://github.com/TradingChassis/docs)
(MkDocs / Material documentation site).

This file is the canonical portfolio analysis record for this project. The
repository remains the technical source of truth. This file is not a CV,
README rewrite, or product page.

Repository analyzed: local clone `main` at **`5420a6b`** (2026-05-25).
No Git tags on `main`. Git authorship: **`bxvtr`** (13 commits).
Remote: `https://github.com/TradingChassis/docs/`.
License: **CC BY 4.0** (`LICENSE`) — a content license, not a software
license.

How to read status labels:

| Label | Meaning in this file |
| --- | --- |
| Implemented Docs Infrastructure | MkDocs config, workflows, requirements, theme overrides |
| Published / Versioned | Present on `gh-pages` via `mike` |
| Documented Architecture | Written as an architecture model in this repo |
| Canonical Terminology | Normative definition in `00-guides/terminology.md` / `20-concepts` |
| ADR / Decision Record | Markdown ADR with Context / Decision / Consequences / Trade-offs |
| Historical | Describes a past design phase |
| Archived | Not the README banner (see below) |
| Architecture Exploration | Design captured as docs; not auto-proof of other repos |
| Cross-repo Reference | Names or paths of other TradingChassis repos |
| Documented but not implementation-verified | Docs claim without this repo proving code |
| Planned | Roadmap / empty placeholders / commented nav |
| Deferred | Evolution docs say not shipped |
| Inferred | Reasonable from multiple artifacts |
| Uncertain | Insufficient evidence |

Source-of-truth for **this repo**:

1. **Docs infrastructure** (MkDocs, workflows, Git, actual Markdown files)
2. **Documentation content** (what was designed/written)
3. **Roadmap / placeholders** (not implemented)

The **legacy banner wins** over present-tense README / `index.md` body
copy (“canonical reference”, “Research-to-Production architecture”).

Classification used throughout:

- **A — Documentation-system fact.** True of this repo’s tooling and files.
- **B — Documented architecture.** True as a written model.
- **C — Implementation claim.** Requires `core`, `core-runtime`, or
  `infrastructure`. **Not** established by this file alone.

---

## Project Status and Historical Context

**Status: Legacy / Architectural Exploration.**

The README opens with:

> This repository is no longer part of the active direction of
> TradingChassis. … TradingChassis has since pivoted away from
> implementing a custom trading engine.

That is **normative**. This repository documents an earlier architecture
phase. It is **not** the current TradingChassis product direction.
Architecture diagrams and Stack pages are **not** implementation proofs
for Kubernetes, Risk Engine, Runtime, storage, or monitoring.

The banner text is **Legacy**, not “Archived.” GitHub Pages still
serves a `mike` version `0.1.0` aliased as `latest`. Treat the site as
a **historically published documentation snapshot**, not a live product
handbook.

Docs-as-Code work here (information architecture, terminology,
ADRs, versioned publishing) is real engineering evidence.

Git: `#11` (`b78c8a8`, 2026-05-19) adds the legacy banner **and**
updates Control-Time / roadmap / milestones text. `#12` (`5420a6b`,
2026-05-25) adds a present-tense positioning table and a high-level
Mermaid workflow to the README **after** the banner. Last `gh-pages`
deploy observed on this clone: **`b78c8a8` → `0.1.0` / `latest`** —
`#12` is on `main` and is **not** in that last Pages deploy.

Evidence:

```text
- README.md (status banner)
- git show b78c8a8
- git show 5420a6b
- git log origin/gh-pages --oneline
```

---

## Project Overview

**Status:** Implemented Docs Infrastructure. Documented Architecture.
Published / Versioned (`0.1.0` + `latest`). Legacy as product direction.

This repository is a **documentation system** for TradingChassis
infrastructure architecture. It is built with MkDocs Material, Mermaid
via `pymdownx.superfences`, `mike` versioning, and a tag / dispatch
GitHub Actions deploy to GitHub Pages.

It does **not** contain trading-engine source, Kubernetes manifests,
or runtime Python packages. Those live in separately inventoried
repos (`TradingChassis/core`, `core-runtime`, `infrastructure`).

Approximate tracked Markdown scale (git `ls-files`):

| Area | Tracked `.md` | Character |
| --- | --- | --- |
| `00-guides` | 6 | 4 substantive + 2 **empty** quickstart files |
| `10-architecture` (excl. ADR bodies counted below) | 8 views + ADR overview | Logical / physical / flows / principles |
| ADRs | 8 | All `Accepted` |
| `20-concepts` | 14 | Canonical semantics |
| `30-stacks` | 42 | 7 stacks × 6-document template |
| `40-operations` | 15 | **1 written** overview + **14 empty** placeholders |
| `50-evolution` | 5 | Roadmap, milestones, 2 dev-log pages, **empty** scaling vision |
| Root | README, CONTRIBUTING, CODE_OF_CONDUCT, SECURITY, `docs/index.md` | |

Mermaid: **32** fences in **27** Markdown files (A).

Evidence:

```text
- mkdocs.yml
- requirements-dev.txt
- .github/workflows/deploy.yaml
- git ls-files '*.md'
```

---

## What I Built

Authorship on `main` is `bxvtr`. This provides strong evidence that I
was the primary author and maintainer of the Docs-as-Code repository,
its architecture corpus, and publishing configuration — not that I
shipped every Stack the site describes.

### Documentation-system work (A)

1. Numbered information architecture (`00` → `50`) with explicit
   reading paths.
2. Canonical terminology page and concept-layer documentation
   governance rules.
3. Eight structured ADRs plus an ADR overview that states role vs
   architecture / concepts / stacks.
4. Repeatable Stack document template applied to seven Stacks.
5. MkDocs Material site: nav tabs, search, admonitions, content tabs,
   dual light/dark logos under `mike` URL prefixes.
6. Mermaid diagrams (architecture map, flows, stack relationships).
7. Exact-pinned Python doc dependencies.
8. `mike` + GitHub Actions deploy to `gh-pages`.
9. CC BY 4.0 content licensing, CoC, docs-only SECURITY.md.
10. Devcontainer stub for local docs work.

### Documented architecture (B)

A two-axis model: **concepts** (what it means) vs **stacks** (how it
would be realized); Event-derived State; layered Strategy / Risk /
Execution Control / Venue Adapter; hftbacktest as simulated Venue
behind an adapter (ADR-008).

### Not this repo (C)

Core library semantics, Runtime orchestration, cluster provisioning,
live trading, production monitoring, Event Stream persistence.

Evidence:

```text
- docs/00-guides/documentation-philosophy.md
- docs/10-architecture/adr/
- docs/30-stacks/stacks-overview.md
- mkdocs.yml
```

---

## Documentation Information Architecture

**Status:** Implemented in the tree. Mostly consistent. Operations and
quickstart are structurally reserved but unpublished / empty.

Numbered prefixes encode reading order (guides → architecture →
concepts → stacks → operations → evolution). `documentation-structure.md`
assigns **role types**: canonical definitions, conceptual bridges,
overviews, applied stack docs, operational docs, decision-history.

| Section | Intended role | What actually exists |
| --- | --- | --- |
| `00-guides` | Orientation, vocabulary | Architecture map, structure, philosophy, terminology. Quickstart files exist and are **empty**; **not in nav**. |
| `10-architecture` | Views + ADRs | Overview, narrative, principles, logical, physical, flows, intent pipeline, 8 ADRs |
| `20-concepts` | Authoritative meaning | Time, Event, State, Determinism, Invariants, Intent/Order lifecycles, dominance, queues, failure, snapshots, ingest vs decision |
| `30-stacks` | Realization, no new semantics | Seven Stacks, six-file template each, in nav |
| `40-operations` | Operate the system | Overview is written but **stale**; remaining files **empty**; **entire nav block commented out** |
| `50-evolution` | History and direction | Roadmap + milestones + Control-Time dev log in nav; `scaling-vision.md` empty and commented out of nav |

`docs/index.md` still advertises Operations, runbooks, and
Research-to-Production as if those pages were a finished handbook.
Nav and empty files contradict that. **Banner + filesystem win.**

Evidence:

```text
- mkdocs.yml (nav, commented Operations)
- docs/00-guides/documentation-structure.md
- docs/index.md
```

---

## Canonical Terminology and Concept Governance

**Status:** Canonical Terminology (documentation convention).
Not technically enforced.

`terminology.md` declares itself the **canonical semantic source**.
Capitalized terms are formally defined; other words are descriptive.
Definitions are **normative for the rest of the documentation**.

Defined terms include (among others): Event, Event Stream, Event Time,
Capture Time, Processing Order, Configuration, State, State Domains,
State Transition, Determinism, Intent, Risk Engine, Queue, Queue
Processing, Execution Control, Order, Core, Strategy, Runtime, Control
Scheduling Obligation, Control-Time Event, Backtesting, Live, Venue,
Venue Adapter, Stack, Canonical Storage.

`documentation-philosophy.md` states governance principles, including:

- **P2 — One concept, one authoritative definition**
- Overviews summarize; they must not add semantics
- Authority order: concept/invariant docs → architecture → overviews
- Semantic drift treated as an architectural regression

These are **process conventions**. There is no linter, link checker, or
CI job that fails a PR for redefining a term in a Stack page.

`CONTRIBUTING.md` is weaker: generic commit/PR advice, “large conceptual
changes should **ideally** be ADRs,” local preview via `mike serve`.
The stronger rules live in philosophy + README, not in automated gates.

Evidence:

```text
- docs/00-guides/terminology.md
- docs/00-guides/documentation-philosophy.md
- CONTRIBUTING.md
```

---

## Architecture Views and ADRs

**Status:** ADR / Decision Record. Documented Architecture.

### Views

Logical architecture, physical deployment topology (recorder/live node,
central cluster, persistent storage), infrastructure flows, intent
pipeline, numbered architecture principles (including **P13**: Core
derives Control Scheduling Obligations; Runtime injects Control-Time
Events). Mermaid is used as **illustration**, not as a deployable
system.

Physical architecture is **B**: a target deployment shape. It does not
prove the `infrastructure` repo provisioned that topology.

### ADR set

**Eight** ADRs, all `adr_status: Accepted` in YAML front matter,
rendered with MkDocs macros (`{{ page.meta.adr_status }}`). Common
sections: **Context, Decision, Consequences, Trade-offs**, plus
Summary. **None marked superseded or deprecated.** Dates cluster on
`2026-04-01` except the Control-Time **dev log** (`In progress`).

| ID | Title | Why it is a strong example |
| --- | --- | --- |
| ADR-001 | Two-axis documentation and architecture | Explicit conceptual vs implementation axis; asymmetric authority; names semantic drift as the failure mode |
| ADR-003 | Event-derived State | Alternatives (mutable state vs Event Stream as truth); `State = f(Event Stream, Configuration)` |
| ADR-005 | Mandatory Risk before Execution Control | Policy vs scheduling split; bypass prohibition |
| ADR-007 | Layered Runtime | Four control questions (what / whether / when / how); feedback only via Events |
| ADR-008 | hftbacktest as simulated Venue | Third-party vs in-house matcher; adapter boundary; fidelity limits |

ADR-002 (Canonical Storage vs Runtime Event Stream) and ADR-004
(derived outbound queue / dominance) and ADR-006 (Intent vs Order
state machines) complete the set. The ADR overview states ADRs record
**why**; they must not duplicate concept definitions.

This is a **small, consistent ADR corpus**, not a large formal ADR
program with IDs, supersession chains, and review workflow beyond GitHub
PRs.

Evidence:

```text
- docs/10-architecture/adr/overview.md
- docs/10-architecture/adr/foundations/ADR-001-two-axis-structure.md
- docs/10-architecture/adr/foundations/ADR-003-event-vs-state-model.md
- docs/10-architecture/adr/execution-engine/ADR-007-layered-runtime-architecture.md
- docs/10-architecture/adr/research-infrastructure/ADR-008-hftbacktest-as-venue.md
- docs/10-architecture/architecture-principles.md
```

---

## Concepts vs Stack Realizations

**Status:** Documented Architecture. Template consistency is strong.
Realization claims are **B**, not **C**.

README and `stacks-overview.md` state: `20-concepts` = what it means;
`30-stacks` = how it is realized **without redefining** semantics.
ADR-001 makes that the architectural rule. Spot-checks of Stack
overviews and implementation-notes generally **apply** terms (Capture
Time, Venue Adapter, Core Runtime) and point back to concepts/ADRs
rather than inventing parallel Event definitions. That split is
**consistently intended** and largely held in the written Stack set.

### Concept documents that exist

Time Model (Event Time vs Processing Order; Control-Time injection;
no hidden wall-clock State), Event Model, State Model, Determinism
Model (`State = f(Event Stream, Configuration)`), Invariants (E\*,
EC\*, lifecycle, risk, determinism groups), Intent lifecycle, Order
lifecycle, Intent dominance, Queue semantics / processing, Failure
semantics, Snapshot-driven inputs, Ingest vs decision frequency.

Prefer: **documented a determinism / time / Event-State model.**
Do **not** write: **implemented a deterministic system** from this
repo alone. Core-runtime inventory already shows FillEvent ingress
and Event Stream persistence as deferred in code.

### Stacks that exist (seven)

Data Recording, Data Quality, Data Storage, Backtesting, Live,
Analysis, Monitoring.

Each has the advertised six files: overview, scope and role,
interfaces, internal structure, operational behavior, implementation
notes. That is evidence of a **repeatable documentation schema**.

There is **no** Stack named Core, Runtime, or Infrastructure. Core
and Runtime appear as **concepts / layers**. Cluster work is
out-of-repo.

Live and data-platform Stack pages describe **target realization
patterns** (feeds, sessions, promotion pipelines). They are not
proof those pipelines ran in production.

Evidence:

```text
- docs/20-concepts/concepts-overview.md
- docs/20-concepts/determinism-model.md
- docs/20-concepts/time-model.md
- docs/30-stacks/stacks-overview.md
- docs/10-architecture/adr/foundations/ADR-001-two-axis-structure.md
```

---

## Operations and Evolution Documentation

**Status:** Operations = Planned / placeholder. Evolution = mixed
historical narrative + present-tense implementation notes about
**other** repos.

### Operations

`mkdocs.yml` comments out the entire Operations nav. Fourteen of
fifteen operations Markdown files are **zero-byte**, including
`runbooks/runbooks-overview.md`, `monitoring.md`, `recovery-model.md`,
`incident-model.md`, and all Backtesting/Live operations pages.

The only written file, `operations-overview.md`, is a **conceptual
runtime model**. It uses older names (“Execution Stack”, “Research
Runtime”) that the Stack taxonomy no longer uses. It links to
Monitoring / Incident / Recovery documents that are empty.

**Do not claim operational runbooks or SRE playbooks.** Those pages
are reserved filenames, not runbooks (no symptoms, commands, rollback,
or ownership).

### Evolution

- `milestones.md` — phased narrative (infra foundation → Core/Runtime
  → formal docs → canonical consolidation → transitional Core/Runtime
  alignment). Mixes **B** with **C**-style statements (package names,
  `python -m core_runtime.local.backtest`, Docker image, Argo/GHCR as
  a “deployment validation track”). Treat C-lines as **cross-repo
  claims to verify elsewhere**. The smoke path cites
  `core_runtime/local/local.json`; the core-runtime inventory uses
  `bt_config_local.json` — a **drift** example.
- `roadmap.md` — directional sequence (harden Core → Runtime →
  research stacks → data platform → analysis → Live). Explicit
  **deferred** list matches the runtime inventory (FillEvent ingress,
  Event Stream persistence, `ProcessingContext`, etc.). Written as
  if the engine direction were still active; **legacy banner wins**.
- Dev log: Control-Time Events, status **In progress**, iterative
  problem → hypothesis style. Complements ADRs; not an ADR itself.
- `scaling-vision.md` — empty, not in nav.

Evidence:

```text
- mkdocs.yml (commented Operations)
- docs/40-operations/operations-overview.md
- docs/50-evolution/roadmap.md
- docs/50-evolution/milestones.md
- docs/50-evolution/dev-logs/execution-control/introducing-control-time-events-for-execution-control.md
```

---

## Docs-as-Code Tooling

**Status:** Implemented Docs Infrastructure.

`mkdocs.yml`:

- Theme: **Material**, `custom_dir: docs/overrides`
- Features: top nav, tabs, instant navigation, expand, sections,
  indexes, path breadcrumbs, integrated TOC, search highlight,
  footer, autohide header
- Palette: default (white) / slate (black) toggle
- Extensions: admonition, attr_list, md_in_html, pymdownx.tabbed,
  snippets (`base_path: docs`), emoji, superfences
- Mermaid: `pymdownx.superfences` custom fence `name: mermaid` with
  `fence_code_format` (Material’s usual client-side Mermaid path —
  not a separate Mermaid plugin package)
- Plugins: `search`, `macros` (ADR front matter)
- `extra.version.provider: mike`
- Custom CSS (`stylesheets/extra.css`: layout, Mermaid overflow,
  typography, dual-logo scheme) and JS
  (`javascripts/disable-nav-drag.js`: prevent dragging nav links)
- Logo partial uses `| url` so assets work under `/latest/` and
  `/0.1.0/` prefixes
- `site_author: bxvtr`; copyright CC BY 4.0; `generator: false`
- **No `site_url` field**

Local commands documented: `mkdocs serve`, `mkdocs build` (**not**
`--strict`). CONTRIBUTING prefers `mike serve`.

Dependencies (`requirements-dev.txt`) are **exact pins**:

```text
mkdocs==1.6.1
mkdocs-material==9.7.1
pymdown-extensions==10.20.1
mike==2.1.3
mkdocs-macros-plugin==1.5.0
```

No markdown lint, spellcheck, or link-checker config in-repo.

Evidence:

```text
- mkdocs.yml
- requirements-dev.txt
- docs/overrides/partials/logo.html
- docs/javascripts/disable-nav-drag.js
- docs/stylesheets/extra.css
```

---

## Versioning and Publishing

**Status:** Published / Versioned with important caveats.

Workflow `.github/workflows/deploy.yaml`:

- Triggers: `push` tags `"*"`, or `workflow_dispatch` with required
  `version` string
- Python 3.11, `pip install -r requirements-dev.txt`
- `permissions: contents: write`
- `mike deploy --push --update-aliases "$VERSION" latest`
- `mike set-default --push latest`
- Checkout `fetch-depth: 0`
- **No** `mkdocs build --strict` job, **no** PR validation workflow

README documents Git tag `0.1.0` (not `v0.1.0`) vs docs version
names, and `gh-pages` as the mike-managed branch.

Observed facts:

- **`main` has no Git tags.** Tag-triggered deploys are configured
  but unused on this clone.
- `origin/gh-pages` `versions.json`: single version **`0.1.0`** with
  alias **`latest`**.
- Multiple gh-pages commits **redeploy the same `0.1.0`** from
  successive `main` SHAs (including `b78c8a8`). That is **mutable
  version reuse**, not an immutable version series.
- Last Pages deploy ≠ current `main` (`#12` missing on Pages).

README terminology link uses
`https://tradingchassis.github.io/docs/latest/...` — a **floating
`latest` URL**. After archive, `latest` can still be overwritten by
another dispatch to `0.1.0`.

Do **not** claim a validated documentation CI pipeline or immutable
doc releases. Claim: **a mike-based Pages workflow exists and has
been used to publish `0.1.0`/`latest`.**

Evidence:

```text
- .github/workflows/deploy.yaml
- README.md (Publishing)
- git log origin/gh-pages
- origin/gh-pages:versions.json
```

---

## Contributor Governance

**Status:** Process convention. Not automated.

| Rule | Where | Enforced? |
| --- | --- | --- |
| Concepts own meaning; stacks must not redefine | README, philosophy, ADR-001, stack overview | Convention only |
| Significant decisions as ADRs | README, CONTRIBUTING (“ideally”) | Convention only |
| Capitalized terms = terminology.md | README, terminology.md | Convention only |
| Small scoped PRs, explain why | CONTRIBUTING | Convention only |
| Build before PR | CONTRIBUTING (`mike serve`) | Not CI |

Community files: Contributor Covenant `CODE_OF_CONDUCT.md`;
`SECURITY.md` scoped to the **documentation website / Actions /
hosting**, not trading systems.

Evidence:

```text
- CONTRIBUTING.md
- docs/00-guides/documentation-philosophy.md
- SECURITY.md
```

---

## Documentation vs Implementation Boundary

This repo **can** evidence architecture writing, terminology
governance, ADRs, Docs-as-Code, and versioned publishing (**A**, **B**).

This repo **cannot alone** evidence:

| Tempting claim | Why not |
| --- | --- |
| Kubernetes was deployed as in physical architecture | `infrastructure` repo |
| Risk Engine / Core reducers exist as specified | `TradingChassis/core` |
| Runtime injects ControlTimeEvent, runs sweeps | `core-runtime` |
| Event Stream persistence / replay | Deferred in evolution docs **and** in runtime inventory |
| Live Stack / data recording ran in production | Stack pages are realization **design** |
| Monitoring and runbooks existed | Empty + commented nav |
| hftbacktest was written here | External library; ADR-008 selects it |

Cross-repo references in this tree are mostly **names and CLI
snippets**, not pinned commit URLs. Drift risk is high (`latest`
docs URL; `local.json` vs actual runtime config filename).

Prefer: **documentation links architecture concepts to
implementation repositories.** Do not re-attribute those
implementations to `docs`.

Evidence:

```text
- docs/50-evolution/milestones.md
- docs/50-evolution/roadmap.md (Deferred explicit)
- docs/30-stacks/live/implementation-notes.md
```

---

## Engineering Decisions and Trade-offs

1. **Two-axis IA (concepts vs stacks)** — reduces semantic drift;
   Stack pages are less self-contained (ADR-001 trade-off, accepted).
2. **ADRs for why, concepts for what** — clean role split; only eight
   ADRs, no supersession history yet.
3. **Repeatable six-file Stack template** — high consistency; volume
   can outrun implementation (Live/data stacks read as target
   architecture).
4. **Operations reserved in the tree but unpublished** — honest
   unfinished ops surface; `index.md` still overpromises it.
5. **mike + `latest` alias, reused `0.1.0`** — simple publishing;
   versions are not immutable snapshots.
6. **Exact pip pins, no `--strict` / link CI** — reproducible tool
   versions; weak doc-quality gate.
7. **Governance in philosophy, not linters** — readable rules; easy
   to violate in a Stack page.
8. **CC BY 4.0 for docs** — correct content licensing vs MIT-style
   code repos.
9. **Legacy banner kept while README `#12` still markets
   Research-to-Production** — historical honesty at the top, leftover
   product voice below.
10. **Control-Time as principle P13 + dev log “In progress”** —
    documents Runtime/Core scheduling split as a model, including
    transitional caveats in evolution docs.

---

## Key Technical Learnings

- Architecture documentation can be treated as **code**: numbered IA,
  templates, ADRs, pinned build tools, versioned Pages.
- **Canonical terminology** only works if other docs refuse to
  redefine it; that is editorial discipline unless CI enforces it.
- Separating **semantic models** from **implementation-facing Stacks**
  is a real technique for multi-repo systems.
- ADRs are most useful when they record alternatives and costs, not
  just the chosen box.
- Cross-repo architecture maps drift when links are `latest` / unpinned
  and when evolution pages copy implementation details.
- Operational documentation is not evidenced by empty runbook files
  or a commented nav section.
- Versioned docs (`mike`) ≠ immutable releases if the same version
  string is redeployed.
- Publishing a **legacy / exploration** banner is itself an
  engineering communication decision: freeze a design direction
  without deleting the corpus.

---

## Historical Evolution and Archive Pivot

| When | What |
| --- | --- |
| `9f6f927` 2026-04-04 | First commit — MkDocs corpus |
| `#1`–`#3` 2026-04-19–22 | Naming, logo, terminology, language |
| `#4` 2026-04-23 | Control-Time Events and Runtime boundary |
| `#5`–`#6` | Rendering fix; Intent lifecycle into Concepts nav |
| `#7` 2026-04-25 | Documentation reorganization |
| `#8`–`#9` | Landing page / index |
| `#10` 2026-05-05 | Spelling |
| `#11` 2026-05-19 | **Legacy banner** + Control-Time/roadmap/milestones edits |
| `#12` 2026-05-25 | README positioning table + workflow Mermaid |

Technical arc: **stand up Docs-as-Code → freeze a canonical model in
prose/ADRs → mark the custom-engine direction legacy → keep adding
README framing.** Same calendar window as Core / core-runtime legacy
banners (2026-05-19), with a README follow-up six days later.

No motivation beyond the written pivot sentence is inferred.

Evidence:

```text
- git log --oneline
- README.md banner
```

---

## Evidence Index

| Area | Paths |
| --- | --- |
| Status | `README.md`, `docs/index.md` |
| IA | `docs/00-guides/documentation-structure.md`, `mkdocs.yml` |
| Terminology | `docs/00-guides/terminology.md` |
| Philosophy | `docs/00-guides/documentation-philosophy.md` |
| ADRs | `docs/10-architecture/adr/` |
| Concepts | `docs/20-concepts/` |
| Stacks | `docs/30-stacks/` |
| Operations placeholders | `docs/40-operations/` |
| Evolution | `docs/50-evolution/` |
| Tooling | `mkdocs.yml`, `requirements-dev.txt`, `docs/overrides/` |
| Publish | `.github/workflows/deploy.yaml`, `origin/gh-pages` |
| Governance | `CONTRIBUTING.md`, `LICENSE`, `SECURITY.md` |

Web search: **not used** for this inventory.

---

## Limitations and Non-Claims

Do not use this repository as evidence for:

- Current TradingChassis product architecture.
- Building or operating the production cluster, Live Stack, or
  monitoring stack described in diagrams.
- A complete Event Stream / fill lifecycle implementation.
- Formal ADR process at org scale (eight Accepted Markdown ADRs).
- Automated terminology or link enforcement.
- Immutable documentation versions (reused `0.1.0`, floating
  `latest`).
- Real runbooks (empty files, nav commented out).
- Authorship of `core`, `core-runtime`, or `infrastructure` code.

Keep:

> Designed and maintained a structured Docs-as-Code architecture
> repository with canonical terminology, ADRs, conceptual models,
> implementation-facing stack documentation, and versioned MkDocs
> publishing — documenting a now-legacy custom-engine architecture,
> not proving that every described Stack was shipped.

---

## Derived Defensible Experience Statements

Valid only with the limits above.

- Designed a numbered documentation IA that separates orientation,
  architecture views, canonical concepts, Stack realizations,
  operations, and evolution, with explicit document-role types.
- Wrote and maintained **canonical terminology** and a documentation
  philosophy that requires Stack pages not to redefine semantics
  (process convention, not a linter).
- Authored a consistent set of **ADRs** (context, decision,
  consequences, trade-offs), including two-axis docs structure,
  event-derived State, Risk vs Execution Control, layered Runtime,
  and hftbacktest-behind-adapter.
- Applied a **repeatable six-document Stack template** across seven
  Stacks as implementation-facing design docs.
- Documented a **determinism and time model** (Processing Order vs
  Event Time, Control-Time injection as Runtime responsibility)
  without claiming the model is fully implemented in code.
- Configured **MkDocs Material**, Mermaid, `mike` versioning, and a
  GitHub Actions deploy to GitHub Pages with exact-pinned
  dependencies.
- Licensed documentation as **CC BY 4.0** and scoped security
  reporting to the docs site rather than a trading system.
- Marked the corpus **legacy architectural exploration** when
  TradingChassis left the custom-engine direction, while leaving
  unfinished operations pages unpublished rather than inventing
  runbooks.

Those statements are invalid if rewritten as “I built the production
TradingChassis platform described in the diagrams,” “canonical docs
prove all stacks shipped,” or “this is the current product architecture.”
