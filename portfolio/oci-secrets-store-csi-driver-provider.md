# OCI Secrets Store CSI Driver Provider — TradingChassis Fork

Evidence-based engineering and learning record for
[`TradingChassis/oci-secrets-store-csi-driver-provider`](https://github.com/TradingChassis/oci-secrets-store-csi-driver-provider).

This file is the canonical portfolio analysis record for this project. The
repository remains the technical source of truth for implementation details.
This file is not a CV, README rewrite, or marketing summary.

Repository analyzed: local clone `main` at `1f9ef4b` (2026-08-10).
Remotes: `origin` → TradingChassis fork; `upstream` →
`oracle/oci-secrets-store-csi-driver-provider`.
License of the application remains Oracle **UPL-1.0**.

How to read status labels:

| Label | Meaning in this file |
| --- | --- |
| Upstream-provided | Present at Oracle `v0.5.0` / merge-base `c828093` |
| TradingChassis-authored | Added on the fork; not in that Oracle baseline |
| TradingChassis-adapted | Upstream file changed for fork CI/build/docs |
| Configured / Integrated | Wiring, overrides, or consumer-side use |
| CI-validated | Exercised by this fork’s GitHub Actions definitions |
| Published | Image publication is evidenced by consumer-side use and a recorded manifest digest; this analysis did not independently re-query GHCR |
| Documented process, not demonstrated | Written procedure; no sync-from-upstream commit on `main` |
| Inferred | Reasonable conclusion from multiple artifacts |
| Uncertain | Insufficient evidence in this clone |

Source-of-truth rule: **the fork diff against Oracle `v0.5.0` wins**.
Presence of Go provider logic, CSI gRPC, OCI Vault SDK usage, or Helm
charts in this tree is **not** TradingChassis authorship.

This record does **not** absorb work from the archived
`TradingChassis/infrastructure-secrets` repository. That line is out of
scope here.

---

## Project Overview

**Status:** TradingChassis-maintained fork of Oracle’s OCI Secrets Store
CSI driver provider. Application code on this clone is **byte-identical**
to local `upstream/main` (`c828093` / Oracle tag `v0.5.0`).

What the **upstream** project is (Oracle):

- A gRPC provider that mounts OCI Vault secrets into pods via the
  Kubernetes Secrets Store CSI Driver.
- Go module
  `github.com/oracle-samples/oci-secrets-store-csi-driver-provider`.
- Helm chart, deploy YAML, E2E examples against OCI/OKE.

What **this fork** adds:

- Multi-architecture container **packaging** (`linux/amd64` +
  `linux/arm64`).
- GitHub Actions that **validate both platforms without publishing** on
  pull requests.
- Publish pipelines to
  `ghcr.io/tradingchassis/oci-secrets-store-csi-driver-provider` using
  `GITHUB_TOKEN` (`packages: write` on publish jobs only).
- Fork documentation (`TRADINGCHASSIS.md`) describing tags, Helm
  overrides, and an upstream-sync procedure.

What this fork is not:

- A rewrite of the OCI Vault / CSI provider.
- An automatic GitHub Release factory.
- Proof that Oracle E2E Vault tests were run by TradingChassis.
- A sync to live Oracle `main` / `v0.5.1` (that object is **not** in
  this clone; infrastructure docs also state PR #60 / `v0.5.1` is
  out of that migration’s scope).

Evidence:

```text
- git merge-base origin/main upstream/main  →  c828093
- git log origin/main --not upstream/main     (exactly two commits)
- git diff --stat upstream/main..origin/main  (14 files, +938/−235)
- TRADINGCHASSIS.md
```

---

## Fork Provenance and Attribution

**Status:** Implemented as a small, reviewable delta on Oracle `v0.5.0`.

| Item | Value |
| --- | --- |
| Upstream | `https://github.com/oracle/oci-secrets-store-csi-driver-provider` |
| Fork | `https://github.com/TradingChassis/oci-secrets-store-csi-driver-provider` |
| Merge-base / local `upstream/main` | `c828093` = Oracle **`v0.5.0`** |
| TradingChassis `main` HEAD | `1f9ef4b` (2026-08-10) |
| TradingChassis commits on `main` | **2** (author `bxvtr`, squash-merged via GitHub) |
| Go / Helm / `deploy/` / `e2e/` / `vendor/` vs merge-base | **no diff** |

Commits on `origin/main` that are not on local `upstream/main`:

| SHA | Date | Subject |
| --- | --- | --- |
| `0d72b51` | 2026-07-24 | `feat(ci): add multi-architecture GHCR publishing (#1)` |
| `1f9ef4b` | 2026-08-10 | `chore(docs): refresh code of conduct and markdown cleanup (#2)` |

Files touched by that range:

```text
A  .github/workflows/ci.yaml
A  .github/workflows/publish-dev.yaml
A  .github/workflows/publish-main.yaml
A  .github/workflows/publish-release.yaml
A  TRADINGCHASSIS.md
A  CODE_OF_CONDUCT.md
D  .github/workflows/build-n-push.yaml      (Oracle)
D  .github/workflows/release.yaml           (Oracle)
M  .github/workflows/e2e-tests.yaml
M  .github/workflows/unit-tests.yaml
M  .golangci.yaml
M  Makefile
M  README.md
M  build/Dockerfile
```

Additional `bxvtr` commits exist only on stale local tracking refs for
deleted topic branches (`feat/multi-arch-ghcr`, `chore/docs-hygiene`).
They are precursors of the two squash merges, not extra `main` work.

**Live Oracle GitHub** (from remote advertisement, not this working
tree): `upstream/main` at analysis time pointed at `v0.5.1`
(`4364088`), which is **not fetched locally**. Attribution of “app code
unchanged” is vs **`v0.5.0`**, not vs today’s Oracle `main`.

Oracle also advertises an unfetched `arm-support` branch. This fork’s
arm64 **image** path is TradingChassis Buildx/CI, not a merge of that
branch.

```text
Evidence:
- git remote -v
- git log origin/main --not upstream/main
- git diff --name-status upstream/main..origin/main
```

---

## What I Changed

**Application / Vault / CSI logic:** unchanged vs Oracle `v0.5.0`.
Do not claim “built the OCI CSI provider” or “implemented OCI Vault
integration.”

**TradingChassis-authored or adapted (the actual fork delta):**

1. **Dockerfile** — `TARGETOS`/`TARGETARCH`, `CGO_ENABLED=0` cross-compile,
   tests **not** run inside the image, OCI labels pointing at the fork.
2. **Makefile** — `docker-buildx`, `PLATFORMS`, `BUILDX_OUTPUT`,
   `--provenance=false`.
3. **CI** — replacement of Oracle `build-n-push` / `release` with
   `ci.yaml` + three publish workflows.
4. **E2E workflow** — converted from PR-triggered (and previously
   chained to image build) to **manual** `workflow_dispatch` with a
   pre-published `image_path`.
5. **Unit-test workflow** — `workflow_call` only; Coveralls upload
   removed; action versions bumped.
6. **golangci-lint config** — golangci-lint v2 schema; enable a small
   correctness set; disable style-only staticcheck `ST*` / `QF*` so
   **unchanged upstream Go** does not fail fork CI.
7. **Docs** — `TRADINGCHASSIS.md`, README fork section, Contributor
   Covenant 3.0 (moved from `docs/` to repo root in `#2`).

**golangci-lint (engineering interpretation):** adapting the linter to
the fork’s Go 1.26 CI without rewriting Oracle source is a
**maintenance** choice: keep application divergence at zero.

**Stale label:** `build/Dockerfile` still sets
`org.opencontainers.image.documentation` to
`.../blob/main/docs/TRADINGCHASSIS.md` after `#2` moved the file to
`TRADINGCHASSIS.md`. Implementation lag, not a second product.

---

## Multi-Architecture Container Build

**Status:** TradingChassis-authored packaging. **Build-validated** in
CI (OCI tarball outputs). Runtime on ARM64 is a **consumer** concern
(see infrastructure), not proven by this repository’s E2E workflow.

### What had to change

Oracle’s image build (baseline) compiled in Docker without Buildx
`TARGETARCH`. Multi-arch on QEMU would either:

- run `make test` under emulation (slow/flaky), or
- produce a binary for the **builder** arch instead of the target.

Fork Dockerfile:

```text
ARG TARGETOS=linux
ARG TARGETARCH
RUN CGO_ENABLED=0 GOOS=${TARGETOS} GOARCH=${TARGETARCH} make build
```

Comment in-tree: unit tests run in `ci.yaml`, not in the image build.

Base images remain **upstream choices**: `golang:1.26` builder,
`oraclelinux:9-slim` runtime. TradingChassis did not replace the OS
base; it parameterized Go’s target arch.

Makefile default registry is still `ghcr.io/oracle-samples` unless
`IMAGE_REGISTRY` is set. CI/publish set
`ghcr.io/tradingchassis` / `tradingchassis/oci-secrets-store-csi-driver-provider`.

`make docker-buildx` builds **one multi-platform image** via Buildx
`--platform=$(PLATFORMS)` (default `linux/amd64,linux/arm64`). Local
default `BUILDX_OUTPUT` is `type=image,...,push=false`. CI uses
`type=oci,dest=/tmp/...` **per architecture, separately** (two jobs),
not a pushed manifest list. Publish workflows use `--push` with both
platforms in one `docker buildx build` so GHCR receives a
**manifest list / OCI index**.

`--provenance=false` is explicit: no BuildKit provenance/SBOM
attestation in these builds.

OCI labels set `vendor=TradingChassis` and source/url to the fork while
the Go module path remains `oracle-samples/...`.

```text
Evidence:
- build/Dockerfile
- Makefile (docker-buildx, PLATFORMS)
- .github/workflows/ci.yaml (docker-amd64, docker-arm64)
- .github/workflows/publish-main.yaml
```

---

## CI and Build Validation

**Status:** TradingChassis-authored orchestration of **upstream** Go
tests plus fork image/Helm checks.

### `.github/workflows/ci.yaml`

Triggers: `pull_request` to `main`; `push` to branches **other than**
`main` and `gh-pages`. **Does not run on merge-to-main** (that is
`publish-main.yaml`).

Permissions: **`contents: read` only** (workflow-level). No
`packages: write`. No GHCR login.

Jobs:

| Job | What |
| --- | --- |
| unit-tests | `go build -mod vendor`; `go test -covermode=count` |
| static-checks | `go vet`; `golangci/golangci-lint-action@v8` (`v2.12.2`) |
| helm | Helm 3.16.4; `dependency update`; `lint`; `template` with **override** `ghcr.io/tradingchassis/...:ci-validate` |
| docker-amd64 | Buildx, `--output=type=oci,dest=/tmp/image-amd64.oci`, `test -s` |
| docker-arm64 | QEMU `arm64` + Buildx, same OCI output pattern |

Actions are pinned by **major/minor tags** (`actions/checkout@v4`,
`setup-go@v5`, `setup-buildx-action@v3`, …), **not** commit SHAs.

Go tests and Helm chart contents are **upstream**. CI **running** them
on PRs, plus no-push multi-arch builds, is TradingChassis work.

### `.github/workflows/unit-tests.yaml`

Reusable `workflow_call` only. Oracle Coveralls step removed. Not the
PR gate (`ci.yaml` duplicates build+test).

```text
Evidence:
- .github/workflows/ci.yaml
- .github/workflows/unit-tests.yaml
- .golangci.yaml
```

---

## GHCR Publishing

**Status:** Implemented in workflows. Consumer infrastructure pins
commit `1f9ef4b` and records digest
`sha256:a04180e28fe6a6b55b1dea934baae174ee1e02ddbb6142157ab706dec0ca180b`
— evidence the SHA tag **existed and was a multi-arch index** at
validation time. This portfolio clone cannot re-query GHCR here.

Canonical name:
`ghcr.io/tradingchassis/oci-secrets-store-csi-driver-provider`

Auth: `docker/login-action@v3` with `username: github.actor`,
`password: secrets.GITHUB_TOKEN`. Docs tell operators to grant the
repo Actions write on the package and to **avoid PATs** for routine
publish. That is configuration guidance plus workflow design, not a
proof of org-level package settings.

Publish jobs add `packages: write` **on the job**, not on PR CI.

No GHA cache, no `docker/metadata-action`, no SBOM upload.

```text
Evidence:
- .github/workflows/publish-main.yaml
- .github/workflows/publish-dev.yaml
- .github/workflows/publish-release.yaml
- TRADINGCHASSIS.md (GHCR permissions checklist)
- TradingChassis/infrastructure argocd + repository-validation.yml
  (consumer; digest check)
```

---

## Tagging and Release Model

**Status:** Implemented in workflow scripts. **No Git tags** exist on
this fork clone; `publish-release.yaml` has **not** been demonstrated
by a `vX.Y.Z` ref here.

### `publish-main.yaml` (push to `main`)

Always considers tags `latest` and `main` (mutable). Adds
`${GITHUB_SHA}` **only if** `imagetools inspect IMAGE:SHA` fails
(tag absent). If the SHA tag already exists, it **refreshes only**
`latest`/`main` and does not overwrite the SHA tag in that branch of
the script.

Then one `docker buildx build --platform=linux/amd64,linux/arm64 --push`
with those tags. Post-push `imagetools inspect` on `main`, `latest`,
and SHA if present.

**Convention vs guarantee:** skipping a push to an existing SHA tag is
a **workflow** immutability attempt. GHCR can still retag if someone
with package write publishes the same tag another way. Not
cryptographic immutability and not `image@sha256:...` deployment.

`CREATED` build-arg is wall-clock. Same Git SHA rebuilt later is
**source-addressable**, not claimed bit-identical.

### `publish-dev.yaml` (`workflow_dispatch`)

Inputs: `ref`, `tag_style` ∈ `{dev, dev-branch, sha-only}`.

Sanitizes branch names to `dev-<slug>` (lowercase, `[^a-z0-9._-]` →
`-`, max 120 chars).

Guards: refuse tags `latest`, `main`, and version-like
`v?N`, `v?N.N`, `v?N.N.N` (SHA compared equal to `git rev-parse HEAD`
is skipped so a rare all-digit hash is not treated as semver). SHA tag
is not overwritten if it already exists; `sha-only` then exits 1.

Does **not** publish semantic release tags from this workflow
(enforced in the tag loop).

### `publish-release.yaml` (push tag `v*.*.*`)

Parses `vMAJOR.MINOR.PATCH` only. Tags image `X.Y.Z` (treated
immutable: **exit 1 if already present**), `X.Y` and `X` (mutable
convenience), and SHA (also fail if SHA tag exists).

Comment in workflow: **GitHub Releases are not created automatically.**

A merge to `main` never invents SemVer. That matches the scripts:
only the release workflow emits `X.Y.Z`.

```text
Evidence:
- .github/workflows/publish-main.yaml (Compute tags)
- .github/workflows/publish-dev.yaml (Guardrails)
- .github/workflows/publish-release.yaml
- TRADINGCHASSIS.md (Image tags table)
- git tag -l  (empty on this clone)
```

---

## Helm Integration and Image Overrides

**Status:** Chart is **upstream-provided** and **unchanged** vs
`v0.5.0`. TradingChassis strategy is **values override**, not a chart
fork rewrite.

DaemonSet template (Oracle):

```text
image: "{{ .Values.provider.image.repository }}:{{ .Values.provider.image.tag | default .Chart.AppVersion }}"
```

`values.yaml` default repository remains
`oci-secrets-store-csi-driver-provider` (short name, not GHCR).
`tag` default empty → chart `appVersion` (`0.10.1` in Chart.yaml —
upstream numbering, not this fork’s Git SHA).

**Why not change chart defaults:** `TRADINGCHASSIS.md` states
intentionally, so upstream Helm syncs stay simpler. Consumer
(`TradingChassis/infrastructure`) deploys the **Oracle** chart at tag
`v0.5.0` and overrides:

```yaml
provider.image.repository: ghcr.io/tradingchassis/oci-secrets-store-csi-driver-provider
provider.image.tag: "1f9ef4b6e123c2914edf842d77483d7ee174bf0a"
```

**Digest form:** because the template inserts a **colon**, a digest
must be split as documented:

```yaml
repository: ghcr.io/tradingchassis/oci-secrets-store-csi-driver-provider@sha256
tag: "<manifest-digest>"
```

That yields `repository@sha256:<digest>`. Infrastructure **does not**
deploy that way; it uses the git-SHA **tag** and verifies the digest
in CI separately. Both facts can be true.

Fork CI `helm template` uses a dummy tag `ci-validate` only to prove
the chart still renders with a GHCR repository override.

```text
Evidence:
- charts/.../templates/provider.daemonset.yaml  (unchanged vs upstream)
- charts/.../values.yaml
- TRADINGCHASSIS.md (Helm usage)
- .github/workflows/ci.yaml (helm job)
- infrastructure/argocd/oci-secrets-app.yaml  (consumer)
```

---

## Supply-Chain and Reproducibility Boundaries

**Implemented in this fork:**

- PR CI cannot push packages (`contents: read`).
- Publish only on trusted refs (`push` to `main`, `workflow_dispatch`,
  version tags) with `GITHUB_TOKEN` + `packages: write`.
- Vendor Go modules (`-mod vendor`) — **upstream** vendoring; CI uses it.
- SHA image tags as **source-addressable** pointers.
- Dev workflow refuses to stamp `latest`/`main`/semver.
- `--provenance=false` (explicitly **no** SLSA provenance blob from
  Buildx).

**Not implemented / do not claim:**

- GitHub Actions pinned to commit SHAs.
- Bit-for-bit reproducible images (timestamps, floating
  `golang:1.26` / `oraclelinux:9-slim` tags, no lockfile for bases).
- Automatic SBOM.
- Kubernetes deploying by digest from this chart without the split
  repository trick.
- Cryptographic tag immutability at the registry.

**Consumer-side (infrastructure, not this repo):** CI pulls the GHCR
token, fetches the manifest for tag `1f9ef4b…`, and asserts digest
`sha256:a04180e…` with platforms amd64+arm64.

```text
Evidence:
- workflow permissions blocks
- Makefile / Dockerfiles --provenance=false
- infrastructure/.github/workflows/repository-validation.yml
  (Verify provider image tag resolves to expected multi-arch digest)
```

---

## Upstream Synchronization Strategy

**Status:** Documented process, **not demonstrated** on `main`.

`TRADINGCHASSIS.md` prescribes: fetch upstream, branch
`chore/sync-upstream-vX.Y.Z` from TradingChassis `main`, merge
`upstream/main`, resolve workflow/Dockerfile/Makefile conflicts, run
CI, PR. Warns against GitHub **Sync fork** after divergence.

`main` history shows **no** merge commit from a later Oracle tag.
Local `upstream/main` is still `v0.5.0`. Live Oracle `v0.5.1` is
unsynced.

Conflict hotspots named in docs match the actual delta: workflows,
Dockerfile, Makefile.

```text
Evidence:
- TRADINGCHASSIS.md (Upstream synchronization)
- git log origin/main --not c828093  (only two TC commits)
```

---

## Integration with TradingChassis Infrastructure

**Status:** Cross-repository **consumer** context. Primary evidence
remains the fork; ARM64 **motivation** is documented in
`TradingChassis/infrastructure`, not in this fork’s first-party files
(no “Ampere” string here).

Infrastructure Argo CD Application:

- Chart source: **Oracle** repo, `targetRevision: v0.5.0`
- Image: TradingChassis GHCR **git-SHA tag**
- Auth values: instance principal enabled; user/workload disabled
- CSI driver: install true; MicroK8s `kubeletRootDir`

`argocd/README.md` (infrastructure): the image exists because the
reference node is **OCI Ampere A1 (ARM64)**; Oracle `v0.5.0` does not
publish a multi-arch artifact.

That is a defensible **integration motive** for the fork, attributed
to the infrastructure docs + the fork’s Buildx platforms, not to
invented comments in the CSI repo.

Do not mix in `infrastructure-secrets` history.

```text
Evidence:
- infrastructure/argocd/oci-secrets-app.yaml
- infrastructure/argocd/README.md (ARM64 / fork provenance)
- infrastructure/.github/workflows/repository-validation.yml
```

---

## Testing and Runtime-Validation Boundaries

| Layer | Whose tests? | What they prove | TradingChassis runtime? |
| --- | --- | --- | --- |
| `go test ./...` | Upstream suite | Provider unit behaviour on CI Ubuntu | CI-validated **build-time** |
| Helm lint/template | Upstream chart + TC override | Chart renders | Build-time |
| Buildx amd64/arm64 OCI files | TC CI | Dockerfile **cross-compiles** | Not a running CSI node |
| GHCR manifest inspect | TC publish + infra CI | Index has two platforms | Not Vault I/O |
| Oracle `e2e-tests.yaml` | Upstream jobs, TC trigger | Would need OCI secrets + OKE | **Manual-only; not evidenced as run** |
| Infra Ansible “DaemonSet Ready” | Consumer | Pod scheduled on their cluster | Separate repo |

E2E changes on the fork: drop `on: pull_request`, drop job `build`
(`build-n-push.yaml` deleted), take `inputs.image_path`,
`permissions: contents: read`. Remaining vault/cluster/kubectl steps
are still Oracle’s workflow body.

**Do not claim** this fork live-validated OCI Vault secret mounts.
Infrastructure may have a Ready DaemonSet; that is consumer
operations, and even then it is not this workflow’s Vault assertion
loop.

```text
Evidence:
- .github/workflows/e2e-tests.yaml (header + on: workflow_dispatch)
- git diff upstream/main..origin/main -- .github/workflows/e2e-tests.yaml
```

---

## Engineering Decisions and Trade-offs

1. **Zero Go application divergence vs Oracle `v0.5.0`.** Fork surface is
   CI/build/docs. Cost: provider fixes still depend on upstream sync or an
   intentionally broader fork delta.
2. **Do not rewrite Helm defaults.** Overrides keep Oracle chart
   identity. Cost: digest deploy is awkward (`repository@sha256` +
   `tag`).
3. **Tests out of Dockerfile.** Makes QEMU arm64 builds tractable.
   Cost: image build does not re-run unit tests (CI job does).
4. **PR CI never publishes.** Trust boundary. Cost: reviewers do not
   get a GHCR preview image from untrusted PRs.
5. **`GITHUB_TOKEN` not PAT.** Least privilege for GHCR from Actions.
   Cost: package must be linked to the repo.
6. **`--provenance=false`.** Simpler/smaller push; no attestation
   chain. Cost: weaker supply-chain story than provenance+SBOM.
7. **SHA tags + mutable `latest`/`main`.** Humans get convenience;
   GitOps can pin SHA. Cost: SHA tag immutability is workflow-level.
8. **SemVer only from Git tags.** Prevents `main` from minting
   versions. Cost: this clone has no release tags yet.
9. **E2E manual.** Avoids burning OCI credentials on every PR. Cost:
   Vault path unproven in this repo’s Actions history.
10. **golangci style exemptions.** CI stays green on Oracle code.
    Cost: style regressions in upstream are not a fork gate.
11. **Actions pinned by tag, not SHA.** Easier to bump. Cost: tag
    mutation risk on GitHub-hosted actions.
12. **Makefile still defaults to `oracle-samples` registry.** Less
    Makefile/upstream conflict. Cost: a naive `make docker-build`
    does not target TradingChassis GHCR.

---

## Key Technical Learnings

Project-specific:

- **A useful fork can be almost only packaging.** Multi-arch GHCR for
  Ampere nodes did not require owning Vault/CSI logic.
- **Buildx `TARGETARCH` is the actual portability fix**; a single
  `go build` in Docker follows the builder unless GOARCH is set.
- **PR CI without `packages: write` is a trust model**, not a missing
  feature.
- **Helm `repository:tag` concatenation is a packaging constraint.**
  Digest pinning then becomes a documented split, or you pin a git-SHA
  tag and verify digest in a different system (infrastructure CI).
- **Immutable tags are a policy** (skip/fail if present) unless the
  registry enforces them.
- **Source-addressable ≠ reproducible image bytes.** `CREATED` and
  floating base tags break byte identity.
- **Keep chart defaults upstream-shaped** to make the next Oracle
  merge about workflows, not values.yaml productization.
- **Turning vendor E2E from PR to `workflow_dispatch`** is how you
  stop leaking cloud credentials into fork PR automation without
  deleting Oracle’s test harness.
- **Linter config can be the compatibility shim** that lets you CI
  unmodified upstream Go.

---

## Historical Evolution

| When | What |
| --- | --- |
| 2023-01 → Oracle `v0.5.0` | Upstream provider, chart, E2E (not TC) |
| 2026-07-24 `0d72b51` | Replace Oracle publish/release Actions; add multi-arch GHCR; Dockerfile/Makefile; TRADINGCHASSIS.md (then under `docs/`) |
| 2026-08-10 `1f9ef4b` | Move CoC and TRADINGCHASSIS.md to repo root; markdown cleanup |
| Infrastructure V2 (separate repo) | Consume SHA-tagged GHCR image on Ampere A1; pin Oracle chart `v0.5.0` |

No later Oracle merge is in this `main` history.

---

## Evidence Index

| Topic | Path |
| --- | --- |
| Fork delta | `git diff upstream/main..origin/main` |
| TC commits | `0d72b51`, `1f9ef4b` |
| Dockerfile | `build/Dockerfile` |
| Buildx | `Makefile` |
| PR CI | `.github/workflows/ci.yaml` |
| Publish | `publish-main.yaml`, `publish-dev.yaml`, `publish-release.yaml` |
| E2E (manual) | `.github/workflows/e2e-tests.yaml` |
| Lint shim | `.golangci.yaml` |
| Fork docs | `TRADINGCHASSIS.md`, `README.md` (fork section) |
| Chart image join | `charts/.../templates/provider.daemonset.yaml` |
| Consumer pin | `infrastructure/argocd/oci-secrets-app.yaml` |

Git used: remotes, merge-base, `log --not upstream/main`, diff
name-status/stat, two squash-merge subjects, empty tag list.
Web search: **not used**.

---

## Limitations and Non-Claims

Do not use this repository as evidence for:

- Authorship of the OCI Secrets Store CSI **provider application**.
- Implementing OCI Vault APIs, gRPC CSI protocol, or Helm chart
  templates (Oracle `v0.5.0`).
- Having synced or tested Oracle `v0.5.1` / `arm-support`.
- Formal reproducibility of container bytes.
- SHA-pinned GitHub Actions.
- Automatic GitHub Releases or any `vX.Y.Z` image from **this clone’s
  tags** (none exist here).
- TradingChassis execution of Oracle OCI/OKE E2E (workflow is
  available, not shown run).
- Production-grade supply chain (no provenance, no SBOM, mutable
  convenience tags).
- Work from `infrastructure-secrets`.

Keep the sentence:

> Maintained a focused fork of Oracle’s provider to publish
> multi-architecture images; did not rewrite the provider.

---

## Derived Defensible Experience Statements

Valid only with the attribution bounds above.

- Maintained a **small-surface fork** of
  `oracle/oci-secrets-store-csi-driver-provider` at **`v0.5.0`**, with
  application/Helm/deploy trees unchanged vs that baseline.
- Extended **container packaging** with Docker Buildx so one pipeline
  produces `linux/amd64` and `linux/arm64` (cross-compile via
  `GOARCH=${TARGETARCH}`, tests kept in CI not in the image).
- Added GitHub Actions **PR CI** that builds both architectures to
  local OCI artifacts **without** GHCR push, using `contents: read`.
- Added publish workflows to GHCR with `GITHUB_TOKEN` and job-scoped
  `packages: write`, tagging `latest`/`main` plus a **git-SHA** tag,
  and refusing to let the **dev** workflow stamp `latest`/`main`/semver.
- Left upstream Helm chart defaults intact and documented **image
  overrides** (including the `repository`+`:`+`tag` digest split).
- Converted Oracle’s PR E2E workflow to **manual** dispatch that
  consumes an already published image path, decoupling cloud E2E from
  pull-request CI.
- Adjusted golangci-lint so fork CI can gate **correctness** linters
  without failing on unchanged upstream style.
- Documented (not yet executed on `main`) an upstream merge workflow
  that avoids GitHub “Sync fork” after divergence.

Those statements are invalid if they are expanded to “I built the CSI
driver” or “I live-validated OCI Vault in this repository’s E2E.”
