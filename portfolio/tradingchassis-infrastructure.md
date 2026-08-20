# TradingChassis Infrastructure

Evidence-based engineering and learning record for
[`TradingChassis/infrastructure`](https://github.com/TradingChassis/infrastructure).

This file is the canonical portfolio analysis record for this project. The repository remains the technical source of truth for implementation details. This file is not a CV, README rewrite, or marketing summary.

Repository analyzed: local clone of `TradingChassis/infrastructure` at
tag/release `v0.2.0` / `main` (`e41bf13`, 2026-08-18).
Git authorship in this repository: all sampled commits are under `bxvtr`.
LICENSE copyright: `bxvtrdev@protonmail.com`.

How to read status labels:

| Label | Meaning in this file |
| --- | --- |
| Implemented | Present in Terraform, Ansible, manifests, tools, tests, or CI |
| Partially implemented | Present, but incomplete or limited by an explicit boundary |
| Historical / deprecated | Version 1 generation; retired from the executable tree |
| Documented as live-proven | Repository runbook/CHANGELOG claim a live OCI rebuild; not re-executed here |
| Documented but stale | Documentation that contradicts later canonical docs or the current tree |
| Planned / deferred | Explicitly not done, or listed as later hardening |
| Inferred | Reasonable conclusion from multiple artifacts, not a single file |
| Uncertain | Conflicting or insufficient evidence |

Source-of-truth rule used here: implementation wins over documentation.
Where they disagree, the disagreement is recorded.

---

## Project Overview

**Status:** Implemented (Version 2 current architecture). Version 1 is historical.

TradingChassis/infrastructure is a **single-node research and backtesting
platform** on Oracle Cloud Infrastructure (OCI). It provisions a reference
Ubuntu ARM host, bootstraps MicroK8s, installs Argo CD, and reconciles a small
set of platform workloads from Git.

Current ownership flow (verified in the tree, not only in README):

```text
Terraform  → OCI network, compute, scratch volume, instance-principal IAM
Ansible    → host baseline, scratch filesystem, MicroK8s, Argo CD bootstrap,
             optional private runtime materialization
Argo CD    → long-lived Kubernetes desired state (apps + oci-secrets)
GitHub Actions → static validation only
```

Canonical operator path (implemented as tools, documented as the Greenfield
runbook):

```text
tools/bootstrap-cloud-shell → tools/deploy-clean-room → tools/verify-clean-room
```

What this project is:

- A reproducible **single-node** MicroK8s environment on OCI Ampere A1.
- An ownership-boundary exercise: cloud provision vs host converge vs GitOps
  vs static CI.
- A clean-room operator workflow with explicit human gates for APPLY, FORMAT,
  and REBOOT.

What this project is not:

- Multi-node Kubernetes, managed Kubernetes (OKE), or an HA control plane.
- A production-grade or enterprise platform claim.
- Application business logic or trade-execution systems.

Evidence:

```text
- README.md
- docs/V2_CLEAN_ROOM_DEPLOYMENT.md
- terraform/*.tf
- ansible/playbooks/site.yml
- ansible/playbooks/private-runtime-config.yml
- argocd/*.yaml
- tools/bootstrap-cloud-shell
- tools/deploy-clean-room
- tools/verify-clean-room
- VERSION_1_BASELINE.md
```

---

## What I Built

Attribution in this repository is strong: Git history is a single-author
sequence from the 2026-02-15 first commit through the 2026-08-18 `0.2.0`
release (`#1`–`#77`). This provides strong evidence that I was the primary
author and maintainer of this repository's implementation and evolution.
Specific files may still contain adapted, generated, or upstream-derived
configuration, so attribution is evaluated at the relevant component level.
It does not mean every upstream chart, image, or CSI provider was written here.

### Version 2 (current) — Implemented

Designed and implemented a layered infrastructure stack:

1. **OCI cloud foundation in Terraform** — VCN, internet gateway, custom route
   table, empty subnet security list, public subnet, compute NSG (SSH ingress
   + all egress), Ampere A1 Flex instance, scratch block volume and
   paravirtualized attachment, instance-principal Dynamic Group, and a
   compartment-scoped `read secret-bundles` IAM policy.
2. **Ansible host bootstrap** — Ubuntu 24 / ARM64 contract checks; fail-closed
   scratch-disk discovery, guarded ext4 format, UUID fstab mount; MicroK8s
   1.29/stable plus addons; UFW host policy; OCI cloud-image firewall
   normalization and a boot oneshot; Argo CD Helm install; GitOps root
   Application apply.
3. **GitOps application layer** — App-of-Apps from `argocd/root-app.yaml`;
   child Applications for postgres, mlflow, monitoring, argo, scratch
   platform/dev/prod, and oci-secrets.
4. **Secret-delivery split** — Argo owns CSI driver + OCI provider; Ansible
   owns SecretProviderClass objects and the MLflow runtime Secret; Vault
   lifecycle and secret **values** stay outside the repo.
5. **Scratch storage contract** — Terraform volume → Ansible `/mnt/scratch`
   → Argo StorageClass + static hostPath PVs + namespace PVCs.
6. **Clean-room operator automation** — Cloud Shell bootstrap, state-aware
   deploy, verification with idempotency and reboot proof.
7. **Static CI** — five GitHub Actions jobs covering safety, security,
   Terraform, Ansible, and GitOps render/contracts. CI does not apply
   infrastructure.

### Related work consumed, not authored in this repo

- Upstream `kube-prometheus-stack` Helm chart (configured, not written).
- Upstream Argo CD / Argo Workflows Helm charts (installed/configured).
- Oracle `oci-secrets-store-csi-driver-provider` chart `v0.5.0` (consumed).
- A **TradingChassis fork image** of that provider for linux/amd64 +
  linux/arm64, referenced by digest in CI and by git-SHA tag in the
  Application. The fork source lives in a different repository.

Evidence:

```text
- git log (bxvtr; first commit 286e5e6 through e41bf13)
- terraform/
- ansible/roles/
- apps/
- argocd/oci-secrets-app.yaml
- .github/workflows/repository-validation.yml
```

---

## Architecture and Ownership Boundaries

**Status:** Implemented as the Version 2 design invariant.

The central engineering product of this repository is not “Kubernetes plus
Terraform.” It is a **one-owner-per-resource** split, enforced in code,
tests, and operator tools.

| Layer | Owns | Does not own |
| --- | --- | --- |
| Terraform | Disposable OCI: VCN, subnet, IGW, NSG, instance, scratch volume/attachment, Dynamic Group, secret-bundle policy | Host config, MicroK8s, Argo CD, Kubernetes manifests, Vault/secret values, Object Storage state bucket |
| Ansible | Host baseline, scratch FS/mount, MicroK8s, UFW + OCI firewall helper, Argo CD **bootstrap**, private SPC/Secret materialization | OCI resources, long-lived app desired state, CSI driver/provider install, MicroK8s Calico UFW interface rules |
| Argo CD | Long-lived Kubernetes desired state, including `oci-secrets`, workloads, scratch SC/PV/PVC | Host mounts, OCI IAM, Vault values |
| GitHub Actions | Static validation of the Git tree | Live provision, converge, or destroy |
| Operator / external | Vault + secret values, state bucket, API signing key, SSH keypair, tenancy/compartment, GitHub repo | Terraform-owned disposable resources after destroy |

Bootstrap vs steady-state (implemented):

```text
Ansible installs Argo CD once and applies argocd/root-app.yaml
→ root Application (namespace argocd) reconciles argocd/ children
→ children reconcile apps/ (and the Oracle chart for oci-secrets)
→ Ansible private-runtime-config.yml then creates SPCs that Git no longer owns
```

`argocd/kustomization.yaml` lists child Applications only. `root-app.yaml` is
applied by Ansible and is **not** included in that Kustomization, so the root
Application does not manage itself.

Steady-state invariant (enforced by overlays + CI): Argo CD and Ansible must
not both be authoritative for the same SecretProviderClass objects. Active
`apps/*/kustomization.yaml` shims point at `overlays/v2`, which omit SPCs.

Defense-in-depth network split (implemented):

```text
OCI NSG          → SSH from operator CIDR; no NodePort ingress
Host UFW         → incoming deny, SSH allow, routed allow
OCI image nft    → FORWARD REJECT removed; narrow pod CIDR INPUT allows
MicroK8s/Calico  → UFW interface allows on vxlan.calico and cali+ (verified, not rewritten)
```

Evidence:

```text
- terraform/network.tf, compute.tf, storage.tf, iam.tf, versions.tf
- ansible/playbooks/site.yml
- ansible/playbooks/private-runtime-config.yml
- ansible/roles/argocd_bootstrap/tasks/main.yml
- ansible/roles/microk8s/tasks/main.yml
- argocd/root-app.yaml
- argocd/kustomization.yaml
- apps/postgres/kustomization.yaml (and mlflow, monitoring)
- AGENTS.md
- docs/V2_CLEAN_ROOM_DEPLOYMENT.md (Ownership model)
```

---

## Cloud Infrastructure

**Status:** Implemented (Terraform root). Live apply/destroy is
**documented as live-proven** in `CHANGELOG.md` `[0.2.0]` and
`docs/V2_CLEAN_ROOM_DEPLOYMENT.md`; not re-run in this analysis.

### What Terraform models

Network (`terraform/network.tf`):

- VCN with configurable CIDR (default `10.0.0.0/16`).
- Internet gateway and a public route table (`0.0.0.0/0` → IGW).
- An **intentionally empty** subnet security list so instance policy lives on
  the NSG, not the VCN default list.
- Public subnet (default `10.0.1.0/24`) with public IPs allowed.
- Compute NSG: TCP/22 from required `ssh_ingress_cidr`; all-protocol egress
  to `0.0.0.0/0` (explicit bootstrap requirement).

Compute (`terraform/compute.tf`):

- Shape `VM.Standard.A1.Flex` (Ampere ARM).
- Image lookup: latest Canonical Ubuntu 24.04 for that shape
  (`sort_by = TIMECREATED`, descending). **Trade-off:** current patches over
  immutable image pinning; later plans can replace the instance.
- Defaults: 4 OCPUs, 24 GB RAM, 50 GB boot volume. Inputs, not a Free Tier
  guarantee.
- Availability domain: first AD returned for the compartment. Acceptable for
  a non-HA reference host; not a multi-AD design.
- Public IPv4 assigned; SSH public key in instance metadata.
- Top-level `is_pv_encryption_in_transit_enabled = true` (not a nested
  `launch_options`-only setting). Live OCI rejected the nested-only form with
  `400-InvalidParameter` unless `NetworkType` was also set (`CHANGELOG`
  `#68`).

Storage (`terraform/storage.tf`):

- Dedicated scratch block volume, default `size_in_gbs = 150`,
  `vpus_per_gb = 0` (Lower Cost).
- Paravirtualized attachment, non-shareable, PV encryption in transit.
- Same AD as the instance.
- **No Linux device path output.** Live paravirtualized attachments did not
  provide a usable `oci_core_volume_attachment.device` (`CHANGELOG` `#53`).

IAM (`terraform/iam.tf`):

- Dynamic Group matching only `instance.id` of the managed node.
- Tenancy policy: `read secret-bundles` in `oci_vault_compartment_id`.
- Vault OCID is an input **reference**, not Vault provisioning.
- Later hardening to `target.secret.id` is documented as not done.

### State, backend, and inputs

- Native `backend "oci" {}` with partial config via gitignored
  `terraform/backend.hcl`.
- State bucket is **external foundation**: not created by this root, must
  survive destroy, Versioning Enabled + NoPublicAccess as operator
  prerequisite.
- Tracked lockfile: `oracle/oci` `8.26.0` for `linux_amd64` and
  `linux_arm64` (CI vs OCI Cloud Shell).
- Auth split: Cloud Shell CLI `instance_obo_user` is **not** a Terraform
  auth mode. Canonical path is APIKey + profile `tradingchassis`.
- CIDR containment validation is hand-written (Terraform 1.15 has no
  `cidrcontains`); covered by
  `tests/unit/test_terraform_cidr_validation.sh` because `terraform validate`
  does not evaluate it against concrete values.

Terraform → Ansible handoff outputs used in practice:
`instance_public_ip` (inventory). Scratch device identity is **not** a
Terraform output.

Evidence:

```text
- terraform/versions.tf
- terraform/providers.tf
- terraform/variables.tf
- terraform/network.tf
- terraform/compute.tf
- terraform/storage.tf
- terraform/iam.tf
- terraform/outputs.tf
- terraform/backend.hcl.example
- terraform/terraform.tfvars.example
- terraform/.terraform.lock.hcl
- tests/unit/test_terraform_cidr_validation.sh
- tests/unit/test_terraform_lockfile_contract.sh
- tests/unit/test_terraform_pv_encryption_contract.sh
```

---

## Configuration Management and Bootstrap

**Status:** Implemented (Ansible). `private_runtime_config` is an explicit
second playbook, not part of `site.yml` (intentional).

Canonical `site.yml` role order:

```text
host_baseline → scratch_storage → microk8s → argocd_bootstrap
  (argocd_bootstrap imports ansible_k8s_runtime)
```

Pinned control-node collections: `ansible.posix 2.2.2`,
`community.general 13.2.0`, `kubernetes.core 6.5.0`. Clean-room tools require
`ansible-core==2.21.2` in `$HOME/.venvs/tradingchassis-ansible`, not Cloud
Shell `/usr/bin/ansible-playbook` (Ansible 2.9).

### host_baseline

Asserts Ubuntu, major version 24, architecture `aarch64`/`arm64`. Does not
configure firewall, packages, or Kubernetes. Provider-neutral on purpose;
OCI shape/image remain Terraform.

### scratch_storage

Fail-closed host-side discovery of the unique eligible non-root whole disk
(`discover_scratch_device.py` over `lsblk` JSON). Guards:

- Candidate must be a block device and must not be the root/boot disk.
- Must not be mounted elsewhere; mount path must be free or already the
  candidate.
- Unexpected existing filesystem type is never converted.
- Blank volume formatting requires `scratch_storage_allow_format=true`
  (default `false`). Discovery never authorizes format.
- Persistent mount by UUID, options `defaults,_netdev,nofail`.
- After mount verification, creates `/mnt/scratch/dev` and
  `/mnt/scratch/prod` (`hostPath` type `Directory` depends on them).

This replaced V1’s hardcoded `/dev/oracleoci/oraclevds`.

### microk8s

- Snap channel `1.29/stable`; addons `dns`, `hostpath-storage`,
  `metrics-server`, `helm` — enable only if `microk8s status --addon`
  reports `disabled`.
- UFW: incoming deny, outgoing allow, routed allow, SSH TCP/22, then enable.
  Does **not** flush iptables (V1 did).
- Does **not** open Kubernetes API, kubelet, NodePorts, or Calico VXLAN to
  the internet via UFW.
- Calico UFW allows on `vxlan.calico` / `cali+` are **MicroK8s-owned**.
  Ansible verifies IPv4 (`ufw-user-*`) and IPv6 (`ufw6-user-*`) persisted
  rules and does not rewrite comments. Dual ownership previously caused
  post-reboot `changed=4` (`CHANGELOG` `#60`, `#61`).
- OCI Ubuntu cloud-image `rules.v4` incompatibilities are handled by
  `normalize_oci_microk8s_firewall.py` plus systemd unit
  `tradingchassis-oci-microk8s-firewall.service`:
  - remove exact IPv4 FORWARD REJECT (blocks Pod → Service forwarding);
  - retain INPUT REJECT;
  - insert pod CIDR `10.1.0.0/16` allows to TCP 16443 and 10250 before it
    (kube-proxy DNAT and metrics-server are INPUT on a single-node node);
  - restore InstanceServices (metadata/DNS/DHCP/iSCSI) without flushing;
  - parse quoted iptables-save comments with `shlex`; compare rules with a
    semantic key so `-p udp --dport 123` vs `-p udp -m udp --dport 123` is
    not drift;
  - `PartOf=` / `WantedBy=` `ufw.service`; `RequiredBy=` MicroK8s
    containerd/kubelite so failed boot reconcile blocks those units.

Live sequence that produced this helper is recorded in `CHANGELOG.md`
`#54`–`#61` (FORWARD, API INPUT, kubelet INPUT, boot reconcile, comment
parse, semantic compare, Calico ownership, IPv6 chain names).

### ansible_k8s_runtime + argocd_bootstrap

Live Ubuntu 24 defects drove the runtime design (`CHANGELOG` `#62`, `#63`,
`#75`):

- Distro `python3-kubernetes` 22.6.0 is below `kubernetes.core` 6.5.0.
- System `python3-pip` / `python3-wheel` was not installable on the image.
- `ansible.builtin.pip` with a control-node `role_path` failed on the
  managed node.

Implemented response:

- Apt-install `python3-venv` only.
- Dedicated venv `/opt/tradingchassis/ansible-kubernetes` with
  `kubernetes==29.0.0`.
- Copy pin file to `/opt/tradingchassis/ansible-k8s-runtime/requirements.txt`
  before remote pip.
- Host roles keep `/usr/bin/python3`.
- Argo CD via Helm chart `argo-cd` `8.2.7`, image tag `v3.0.23` (K8s 1.29
  still in that tested matrix; V1’s `v3.3.0` pin was not kept).
- Helm timeout `"10m"` (Helm 3 duration). Bare integer `600` became
  `--timeout 600` and failed on Helm 3.9.
- Declarative `kustomize.buildOptions: --enable-helm`.
- Apply `argocd/root-app.yaml` into namespace `argocd`.
- Does not reproduce V1 CRD deletion or live Application patching.

### private_runtime_config

Separate playbook. Fail-closed Vault OCID shape
`ocid1.vault.oc1.<region>..<unique>` (empty future-use component), region
match, placeholder rejection, `no_log: true` on private values.

Bounded condition waits (~6 minutes) **before** mutation:

- SecretProviderClass CRD
- CSIDriver `secrets-store.csi.k8s.io`
- DaemonSets in `kube-system` matching the Oracle chart render
- Namespaces `postgres`, `mlflow`, `monitoring` (Argo `CreateNamespace`)

Then materializes:

- Secret `mlflow/tradingchassis-runtime-config` key `OCI_REGION`
- Three SecretProviderClass objects, `authType: instance`, literal `vaultId`

Does not prove Vault retrieval, CSI mount success, or secret contents.

**Documentation inconsistency:** `ansible/README.md` still says the live
sequence “has **not** yet been proven” and that scratch binding “live
clean-room validation remains open.” Canonical `README.md`,
`docs/V2_CLEAN_ROOM_DEPLOYMENT.md`, and `CHANGELOG.md` `[0.2.0]` record
Greenfield acceptance as live-proven and state that trailing “not yet live
proven” Fixed-entry phrases are superseded. Treat `ansible/README.md` as
**stale Level 2** on those sentences.

Evidence:

```text
- ansible/playbooks/site.yml
- ansible/playbooks/private-runtime-config.yml
- ansible/roles/host_baseline/
- ansible/roles/scratch_storage/
- ansible/roles/microk8s/
- ansible/roles/ansible_k8s_runtime/
- ansible/roles/argocd_bootstrap/
- ansible/roles/private_runtime_config/
- ansible/roles/scratch_storage/files/discover_scratch_device.py
- ansible/roles/microk8s/files/normalize_oci_microk8s_firewall.py
- ansible/roles/microk8s/templates/tradingchassis-oci-microk8s-firewall.service.j2
- tests/unit/test_ansible_*.sh
```

---

## Kubernetes and GitOps

**Status:** Implemented (Version 2 overlays active). Version 1 overlays and
Bash bootstrap are **removed** (`tests/unit/test_retired_v1_paths_contract.sh`).

### Argo CD Applications

All child Applications use `repoURL: https://github.com/TradingChassis/infrastructure`,
`targetRevision: main`, automated `prune` + `selfHeal`.

| Application | Path | Namespace | Notes |
| --- | --- | --- | --- |
| `root` | `argocd` | `argocd` | Applied by Ansible; not in child Kustomization |
| `oci-secrets` | Oracle chart `v0.5.0` | `kube-system` | sync-wave `-1`; `CreateNamespace=false` |
| `postgres` | `apps/postgres` | `postgres` | sync-wave `1` |
| `mlflow` | `apps/mlflow` | `mlflow` | sync-wave `1` |
| `monitoring` | `apps/monitoring` | `monitoring` | sync-wave `1`; `ServerSideApply=true` |
| `argo` | `apps/argo` | `argo` | default wave |
| `scratch-storage` | `apps/scratch/platform` | `kube-system` placeholder | cluster-scoped SC/PVs |
| `scratch-dev` / `scratch-prod` | PVC only | `dev` / `prod` | `CreateNamespace=true` |

Sync-wave orders **Application CR create/sync**, not child health. Runtime
readiness for secrets is Ansible’s bounded waits.

Monitoring SSA exists because Prometheus Operator CRDs exceeded the
262144-byte last-applied annotation (`CHANGELOG` `#67`). CI asserts those
CRDs still exceed the limit so SSA is not dropped accidentally.

### Helm vs Kustomize (what was configured here)

- **Kustomize shims + overlays:** postgres, mlflow, monitoring, scratch.
- **Helm-through-Kustomize:** `kube-prometheus-stack` `72.6.2` and
  `argo-workflows` `0.47.1` via `helmCharts` + values files. `--enable-helm`
  is required.
- **Helm via Argo Application `helm.valuesObject`:** `oci-secrets` (Oracle
  chart, not vendored in this repo).
- **Helm via Ansible:** Argo CD itself (bootstrap only).

Vendored chart tree under `apps/monitoring/base/charts/` is upstream chart
content, not original Prometheus/Grafana code.

### Workload manifests (authored)

PostgreSQL (`apps/postgres/base/`):

- Deployment `postgres:14`, env from `postgres-secret`, CSI mount of
  `postgres-secret-bundle`, PVC `postgres-pvc` on **`microk8s-hostpath`**
  (intentionally **not** the scratch contract), Service TCP 5432.
- Job `init-mlflow-postgres`: CSI mount so the Job can sync secrets itself;
  `backoffLimit: 30`, `activeDeadlineSeconds: 1800`; bounded `pg_isready`;
  idempotent role/database create. Designed for delayed SPC availability.

MLflow (`apps/mlflow/`):

- Deployment `ghcr.io/mlflow/mlflow:v3.0.0rc2`; CSI `mlflow-secret-bundle`;
  `MLFLOW_BACKEND_STORE_URI` from Secret; V2 patch adds
  `AWS_DEFAULT_REGION` from `tradingchassis-runtime-config` / `OCI_REGION`.
- Runtime `pip install psycopg2-binary==2.9.9` inside the container command.
- NodePort `30500`. Resource requests/limits set. **No probes.**

Monitoring extra objects:

- Pushgateway Deployment `prom/pushgateway:v1.6.2` + ClusterIP Service.
- Helm values: Grafana/Prometheus NodePorts `30007` / `30090`; Grafana
  admin from `monitoring-secret`; CSI extra volume; scrape config for
  pushgateway; Prometheus Operator admission webhooks/TLS disabled.

Argo Workflows:

- Server NodePort `32120`, `authMode: server`.
- Controller uses existing SA `argo-workflow-sa`; RoleBinding to namespaced
  Role `admin` (namespace-admin, not cluster-admin).

**Not present in authored app YAML:** liveness/readiness/startup probes
(postgres, mlflow, pushgateway). Image pins are tags, not digests, except
the OCI provider tag which CI resolves to a recorded multi-arch digest.

Evidence:

```text
- argocd/*.yaml
- apps/postgres/base/*
- apps/mlflow/base/* and overlays/v2/aws-default-region-patch.yaml
- apps/monitoring/base/helm-values.yaml, deployment.yaml, service.yaml
- apps/argo/*
- tests/unit/test_monitoring_server_side_apply_contract.sh
- .github/workflows/repository-validation.yml (GitOps job)
```

---

## Storage

**Status:** Implemented end-to-end in Git. Host mount + PVC binding are
**documented as live-proven** by clean-room verify; `ansible/README.md`
still says live binding “remains open” (stale).

V2 contract:

```text
Terraform  → OCI block volume (default 150 GB) + paravirtualized attachment
Ansible    → discover disk, optional FORMAT, UUID mount /mnt/scratch,
             create /mnt/scratch/dev and /mnt/scratch/prod
Argo CD    → StorageClass tradingchassis-scratch (no-provisioner, Retain)
             static hostPath PVs + namespaced PVCs with volumeName + claimRef
```

Kubernetes accounting: `70Gi + 70Gi = 140Gi` against 150 OCI-GB, headroom
for filesystem overhead. PV capacity is **not** a quota.

`hostPath.type: Directory` adds a storage-safety check: missing subdirectories
after a bad remount cause the volume mount to fail instead of silently creating
unexpected paths.

PostgreSQL stays on `microk8s-hostpath` (boot disk). That is an explicit
non-goal of the scratch contract, checked in CI.

V1 gap (historical): host mounted `/mnt/scratch` while scratch PVCs used
`microk8s-hostpath` and did not demonstrably use that mount. Closed in V2
Git manifests.

Evidence:

```text
- terraform/storage.tf
- ansible/roles/scratch_storage/tasks/main.yml
- apps/scratch/platform/storageclass.yaml
- apps/scratch/platform/pv-dev.yaml
- apps/scratch/platform/pv-prod.yaml
- apps/scratch/dev/pvc.yaml
- apps/scratch/prod/pvc.yaml
- apps/postgres/base/pvc.yaml
```

---

## Secrets and Security

**Status:** Implemented for delivery/references/IAM/CI hygiene. Vault
lifecycle and secret **values** are out of scope.

### Secret architecture

```text
External OCI Vault (secret names must exist)
→ Terraform instance principal (Dynamic Group + read secret-bundles)
→ Argo oci-secrets: CSI Driver 1.3.3 + OCI provider (fork image, instance auth)
→ Ansible SecretProviderClass (vaultId, secret name → fileName → Secret keys)
→ workload secretKeyRef / Grafana existingSecret
```

`authType: instance` everywhere. User and workload auth types disabled in
Helm values. MicroK8s kubelet root is set to
`/var/snap/microk8s/common/var/lib/kubelet` (not `/var/lib/kubelet`).

Provider image:

```text
ghcr.io/tradingchassis/oci-secrets-store-csi-driver-provider:1f9ef4b6e123c2914edf842d77483d7ee174bf0a
```

The Oracle chart renders `repository:tag`, not `image@sha256`. CI pulls the
GHCR manifest list and requires digest
`sha256:a04180e28fe6a6b55b1dea934baae174ee1e02ddbb6142157ab706dec0ca180b`
with `linux/amd64` and `linux/arm64`. Kubernetes does not deploy by digest.

Required Vault **names** (values never in Git): `postgresdb-naming`,
`postgres-user`, `postgres-password`, `mlflowdb-naming`, `mlflow-user`,
`mlflow-password`, `mlflow-db-uri`, `grafana-login-user`,
`grafana-login-password`.

Private inputs are gitignored: `terraform/backend.hcl`,
`terraform/*.tfvars`, `ansible/extra-vars/private-runtime.yml`,
`ansible/inventory/local.yml`. Examples stay placeholders.

### Network access model

Terraform NSG allows SSH from operator CIDR only. NodePorts exist in
manifests for SSH local forwarding. Cloud-firewall/NSG does not publish
30007/30090/30500/32120. UFW also does not open NodePorts.

`SECURITY.md` states SSH-only public reachability and “all Kubernetes
service ports are blocked at the cloud firewall.” NSG implementation
supports that **by omission** (no NodePort rules). Effective tenancy NSGs
outside this root are Uncertain.

### Repository security CI

- Pinned official Gitleaks CLI (not `gitleaks-action`; org license avoided).
- Tree scan + git history scan (`fetch-depth: 0`; PRs scan `base..head`).
- `tools/check-sensitive-metadata` for operator homes, live-looking OCIDs,
  likely public IPs — distinct from credential scanning.
- Tests must use synthetic fixtures, not copied live values.
- Documented incident rule: if a real credential is in Git, rotate first;
  deleting from `HEAD` is insufficient.

### Agent / operator guardrails

`AGENTS.md`, `.cursor/rules/safe-infrastructure-development.mdc`, and
`docs/AI_AGENT_WORKFLOW.md` forbid live OCI/Terraform/Ansible/K8s from
implementation agents, require evidence classification, and separate
implementation vs PRE-COMMIT review vs authorized finalization. These are
steering, not a sandbox.

`SECURITY.md` also says “infrastructure changes are fully declarative.”
That is stronger than the implementation: Ansible owns runtime SPCs that
are not Git desired state, and clean-room tools are imperative wrappers
around Terraform/Ansible. Prefer the ownership table above.

Evidence:

```text
- terraform/iam.tf
- argocd/oci-secrets-app.yaml
- ansible/roles/private_runtime_config/
- ansible/roles/private_runtime_config/templates/secret-provider-class.yaml.j2
- .gitignore
- .gitleaks.toml
- docs/REPOSITORY_SECURITY.md
- SECURITY.md
- tools/check-sensitive-metadata
- tools/check-agent-safety
```

---

## Observability and Platform Services

**Status:** Implemented as GitOps configuration of upstream software.

### Observability (configured, not authored)

`kube-prometheus-stack` 72.6.2 via Kustomize Helm:

- Prometheus and Grafana as NodePort services.
- Grafana credentials from CSI-synced `monitoring-secret`.
- Additional scrape job for in-cluster Pushgateway.
- Operator admission webhooks disabled (single-node simplification).
- Server-Side Apply on the Argo Application so large CRDs can sync.

Authored extra: Pushgateway Deployment/Service.

This demonstrates Helm values, CSI volume injection into Grafana, Argo
sync-option selection, and CRD apply-size debugging. It does not demonstrate
authoring Prometheus, Grafana, or alerting rule packs.

### Platform services (deployment work)

| Component | Work in this repo |
| --- | --- |
| PostgreSQL | Manifests, CSI secret mount, hostpath PVC, MLflow init Job |
| MLflow | Manifests, CSI URI, region Secret patch, NodePort, resource limits |
| Argo Workflows | Helm values, SA, namespace-admin RoleBinding, NodePort |
| Scratch | SC/PV/PVC split across three Applications |
| oci-secrets | Argo Helm values, MicroK8s kubelet path, instance auth, image pin |

MLflow image is an RC tag (`v3.0.0rc2`). Postgres image is floating
`postgres:14`. Both are implemented facts, not production hardening.

Evidence:

```text
- apps/monitoring/base/kustomization.yaml
- apps/monitoring/base/helm-values.yaml
- apps/monitoring/base/deployment.yaml
- argocd/monitoring-app.yaml
- apps/postgres/, apps/mlflow/, apps/argo/, apps/scratch/
```

---

## Deployment Automation

**Status:** Implemented. Greenfield path **documented as live-proven**
(`CHANGELOG` `[0.2.0]`, runbook “Live acceptance status”). Destroy is a
documented operator Terraform procedure, **not** a repo tool.

### tools/bootstrap-cloud-shell

Prepares an already-cloned control node. Does not clone, deploy, overwrite
completed operator files, or start tmux.

- User-local Terraform `1.15.8` under `$HOME/bin` from HashiCorp releases,
  SHA256SUMS verified; refuses incompatible existing binary; maps
  aarch64→arm64 / x86_64→amd64.
- Python 3.12 venv with pinned ansible-core + Galaxy collections from
  `ansible/requirements.yml`.
- Copies `backend.hcl.example`, `terraform.tfvars.example`,
  `private-runtime.yml.example` only if destinations are absent (`chmod 600`).

### tools/deploy-clean-room

State-aware. No internal step-state file. Does not reboot. Does not use
`-auto-approve`.

1. Strict Cloud Shell readiness.
2. Reject angle-bracket placeholders in gitignored inputs.
3. Read-only `oci iam region list --auth api_key` preflight.
4. `terraform init -backend-config=backend.hcl`, validate, plan to a saved
   file.
5. Plan `0` → skip APPLY. Plan `1` → STOP. Plan `2` → JSON inspect; **STOP
   on any `delete` (including replace)**; else exact `APPLY`.
6. Post-apply must be no-change before Ansible.
7. `tools/render-ansible-inventory` (public IP only; no device path).
8. Bounded SSH wait; `site.yml`.
9. FORMAT offered **only** if the log contains the exact blank-scratch
   assertion message. Unrelated failures do not offer FORMAT.
10. `private-runtime-config.yml`.
11. Bounded Argo wait: every Application Synced+Healthy. Missing/Degraded/
    Progressing WAIT inside the window; empty/malformed JSON FAIL.
12. Bounded pod wait: Succeeded Jobs OK; CrashLoopBackOff-class FAIL.

Resumability is actual Terraform/Ansible/Kubernetes state (documented live
resume after PR `#75`).

### tools/verify-clean-room

Never applies Terraform, never formats scratch.

Pre-reboot: no-drift plan, SSH, scratch mountpoint, Argo, workloads,
`private-runtime-config.yml` and `site.yml` with PLAY RECAP `changed=0`.
Exact `REBOOT`. Post-reboot: SSH drop/return, **boot_id must change**,
MicroK8s ready, scratch mount, Argo, workloads.

### Supporting tools

- `tools/render-ansible-inventory` — read-only Terraform outputs → ignored
  inventory.
- `tools/check-cloud-shell-readiness` — local/static; `--strict` on deploy.
- `tools/validate-safe` / `tools/check-agent-safety` — no live infra.
- Shared `tools/lib/clean-room-common.sh` — plan destructiveness, Argo/pod
  JSON evaluators, PLAY RECAP parser, placeholder detection.

Destroy acceptance (runbook only): inspect `state list`, saved destroy plan
must contain only disposable root resources, apply, prove empty state, prove
fresh plan is create/add-only, prove Vault and state bucket survived.
Resource counts are taken from state, not a hard-coded “N destroys.”

Evidence:

```text
- tools/bootstrap-cloud-shell
- tools/deploy-clean-room
- tools/verify-clean-room
- tools/lib/clean-room-common.sh
- tools/render-ansible-inventory
- tools/check-cloud-shell-readiness
- tests/unit/test_clean_room_automation_contract.sh
- tests/unit/test_cloud_shell_execution_contracts.sh
- docs/V2_CLEAN_ROOM_DEPLOYMENT.md
```

---

## Reliability and Operational Engineering

**Status:** Implemented in automation and Ansible; live reboot/idempotency
**documented as live-proven** for the Greenfield path.

Concrete reliability work in this repo (not generic SRE slogans):

| Topic | What exists |
| --- | --- |
| Idempotency | Verify requires second `site.yml` and `private-runtime-config.yml` `changed=0`; firewall helper and scratch role are written for no-op converges |
| Fail-closed mutation | Blank scratch format, destructive Terraform plans, unexpected firewall files, Vault OCID shape, dual SPC ownership |
| Human gates | Exact `APPLY` / `FORMAT` / `REBOOT`; any other input STOP |
| Resumability | No custom step machine; resume from real state (including after `#75`) |
| Reboot recovery | systemd oneshot after UFW; boot_id change proof; scratch UUID mount `_netdev,nofail`; MicroK8s RequiredBy firewall unit |
| Convergence waits | Bounded Argo/pod/CSI/namespace waits; Degraded is WAIT not instant fail during first sync (`#71`, `#72`) |
| Ownership races | Calico UFW left to MicroK8s; SPC Git vs Ansible exclusive ownership |
| CI vs live | CONTRIBUTING and runbook: green Actions ≠ live rebuild |
| Debugging from live infrastructure failures | Device path, CIDR validation, PV encryption create arg, Cloud Shell auth, FORWARD/INPUT nft, Helm timeout, pip path, Vault OCID double-dot, SSA CRDs, system pip/wheel |

This is the strongest SRE-adjacent evidence in the repository: a series of
live defects, each converted into a fail-closed contract plus a static
regression test, then folded into the operator path.

Evidence:

```text
- CHANGELOG.md [0.2.0] Fixed / Changed
- tools/verify-clean-room
- ansible/roles/microk8s/
- ansible/roles/scratch_storage/
- tests/unit/test_ansible_oci_forward_reject_contract.sh
- tests/unit/test_ansible_microk8s_calico_ufw_ownership.sh
- tests/unit/test_ansible_scratch_device_discovery_contract.sh
```

---

## CI and Validation

**Status:** Implemented. Static only.

Workflow: `.github/workflows/repository-validation.yml`
(`Repository Validation`). Permissions: `contents: read`. Five jobs,
documented as GitHub ruleset `main-protection` required checks (the ruleset
itself is **external GitHub config**, not a file in this repo).

| Job | What it proves | What it does not prove |
| --- | --- | --- |
| Static repository checks | Agent-safety scripts, unit contracts, `validate-safe`, ShellCheck | Live hosts |
| Security validation | Gitleaks dir+git; metadata hygiene | Token validity; live IAM |
| Terraform validation | `fmt`, backend contract Python, Cloud Shell contracts, CIDR/lockfile/PV-encryption unit tests, `init -backend=false`, `validate` with synthetic TF_VARs | OCI create, remote state write, image/AD data sources at apply time |
| Ansible validation | ansible-lint, syntax-check both playbooks, SPC/overlay contracts | Host converge, idempotency |
| GitOps validation | `kustomize build` argocd + apps; scratch binding; Helm template of fetched Oracle chart; CSI name cross-check vs Ansible defaults; GHCR digest/platforms; V2 overlays have no SPCs; postgres Job bootstrap contract | Cluster sync, CSI mount, Vault read |

Additional local unit tests cover retired V1 paths, clean-room tool
contracts, and monitoring SSA.

`CONTRIBUTING.md`: a green workflow is not live infrastructure validation.
Operator deploy must not use `terraform init -backend=false`.

Evidence:

```text
- .github/workflows/repository-validation.yml
- tests/unit/*
- CONTRIBUTING.md
- docs/REPOSITORY_SECURITY.md
```

---

## Engineering Decisions and Trade-offs

These are visible in the tree, not inferred from slogans.

1. **Single-node MicroK8s instead of OKE / multi-node.**
   Keeps ownership and cost clear for research. Gives up HA, managed control
   plane, and multi-AZ. Documented as intentional.

2. **Terraform does not own the state bucket or Vault.**
   Avoids circular bootstrap and accidental destroy of foundation. Operator
   must create the bucket and secrets out of band.

3. **Cloud Shell APIKey vs instance_obo_user.**
   Native Terraform OCI backend/provider do not document Cloud Shell
   delegation-token auth. Extra operator key material in exchange for a
   supported, preflightable identity.

4. **Latest Ubuntu platform image vs pinned image OCID.**
   Security updates vs possible instance replacement on later plans.

5. **Paravirtualized scratch vs iSCSI.**
   Simpler on ARM A1; loses a stable Terraform device-path handoff. Ansible
   discovery is the compensation.

6. **UFW + surgical nft helper vs V1 iptables flush.**
   Preserves OCI InstanceServices and INPUT REJECT. Cost: a custom Python
   helper, boot unit, and several live parsing/idempotency bugs.

7. **Ansible SPCs vs Git-owned SPCs.**
   Keeps `vaultId` out of Git. Cost: runtime objects not fully represented
   in Git; dual-ownership must be prevented; `site.yml` does not auto-run
   private-runtime (ordering after CSI).

8. **Helm-through-Kustomize and Helm-through-Argo mixed.**
   Workloads in-repo; Oracle chart fetched by Argo at `v0.5.0`. Provider
   image comes from a TradingChassis fork because upstream v0.5.0 is not
   multi-arch.

9. **Server-Side Apply only on monitoring.**
   Minimal change for oversized CRDs. Other apps stay client-side.

10. **NodePort + SSH tunnel vs Ingress / public HTTPS.**
    Smaller public surface; no TLS ingress in this repo.

11. **Static hostPath PVs vs dynamic provisioner.**
    Deterministic bind to `/mnt/scratch/{dev,prod}` on one node. Not
    portable to multi-node without redesign.

12. **Tag-deployed provider image + CI digest check vs `image@sha256`.**
    Chart limitation vs supply-chain pinning in GitOps.

13. **Imperative clean-room tools around declarative layers.**
    Needed for gates, plan inspection, and Cloud Shell facts. Risk of a
    second control plane is avoided by not persisting step state.

14. **PostgreSQL on hostpath, scratch on dedicated volume.**
    Scratch is research data with Retain semantics; postgres size is 5Gi on
    the boot disk. Different failure domains, not a single storage class.

---

## Key Technical Learnings

Formulated from work in this repository, not textbook definitions.

- **Ownership boundaries are operational, not diagram-only.** When Ansible
  rewrote MicroK8s Calico UFW comments, verify failed `changed=0` even
  though the firewall was correct. The fix was to stop owning those rules.
- **Provisioning ≠ configuration ≠ desired state.** Terraform can attach a
  volume and still not know `/dev/sdb`. Kubernetes can declare a PVC and
  still miss the host mount (V1). Each layer needs an explicit contract.
- **GitOps bootstrap is a special case.** Someone must install Argo CD
  before Argo CD can install everything else. That someone (Ansible) must
  then stop owning long-lived objects.
- **Secrets in Git vs secrets in cluster vs secrets in Vault** are three
  different things. Removing `${VAULT_ID}` from Git did not make Ansible
  the Vault owner; it made Ansible the SPC owner.
- **Instance principal is IAM design, not a CSI checkbox.** Dynamic Group
  match on one instance.id plus compartment-scoped `read secret-bundles` is
  the policy this architecture can actually implement without secret OCIDs
  in Git.
- **Idempotency is a compare function.** nft rendering `-m udp` looked like
  drift until comparison used a semantic key. Calico comment rewrites looked
  like drift until ownership moved.
- **Reboot is a different code path from converge.** Persistent `rules.v4`
  survived; runtime nft did not, because UFW initializes first. A oneshot
  after `ufw.service` was required.
- **Static CI will not see live OCI quirks.** CIDR `cidrcontains`, empty
  attachment.device, LaunchOptions NetworkType, Helm duration units, apt
  wheel, Vault OCID double-dot, CRD annotation size — all appeared at apply
  or converge time. The engineering response was: encode the live lesson as
  a fail-closed contract and a unit test that does not need OCI.
- **Clean-room proof is a procedure, not a green check.** Destroy that
  leaves Vault and the state bucket intact is part of that proof, and it is
  deliberately not automated in CI.
- **Control-node vs managed-node paths matter.** Pip using `role_path` is
  correct on paper and wrong when pip runs on the host. Cloud Shell Python
  3.9/Ansible 2.9 vs required 3.12/2.21.2 is the same class of bug.
- **Fail-closed formatting is a product decision.** Automatic discovery of
  a blank disk is not authorization to `mkfs`.

---

## Historical Evolution

Do not describe V1 and V2 as concurrent executable architectures.

### Version 1 — Historical / removed

Recorded in `VERSION_1_BASELINE.md` and `CHANGELOG.md` `[0.1.0]` (2026-07-30).
Executable paths retired in `#73` (`scripts/`, `apps/*/overlays/v1`,
`infrastructure/oci-provider`). Absence is CI-enforced.

V1 solved “bring up a usable single-node GitOps cluster on a **prepared**
Ubuntu VM,” not “provision OCI as code.”

| Area | V1 owner |
| --- | --- |
| VM, network, volume, Vault, IAM | Manual / external |
| Host bootstrap | Bash `scripts/01`–`07` |
| iptables | Flush + default ACCEPT |
| Scratch device | Hardcoded `/dev/oracleoci/oraclevds`; format if empty |
| CSI + OCI provider | Bash + in-repo manifests |
| Argo CD | Bash install, often discussed around namespace `default` |
| Runtime Vault/region | Live Application patches (`inject-runtime-values.sh`) |
| Scratch PVCs | `microk8s-hostpath`, not bound to `/mnt/scratch` |
| CI | Not present in that generation |

Known V1 limits that V2 was designed to close: non-idempotent bootstrap,
firewall wipe, Git/cluster drift from live patches, storage mount/PVC gap,
no Terraform/Ansible, no static CI.

### Version 2 — Current

Built incrementally on `main` (PRs `#27`–`#77` in 2026-08): Terraform
foundation → network/compute/storage/IAM → Ansible roles → Argo oci-secrets
→ private-runtime playbook → V2 overlays → Cloud Shell/state → live defect
fixes → clean-room tools → V1 path retirement → Greenfield acceptance
writeup → `0.2.0`.

`docs/RUNTIME_SPC_OWNERSHIP_CUTOVER.md` is an **in-place fallback** for
moving SPCs from Argo to Ansible on an existing cluster. It is not the
Greenfield path and is not claimed live-validated by that document.

---

## Evidence Index

| Claim area | Primary paths |
| --- | --- |
| Current architecture statement | `README.md`, `docs/V2_CLEAN_ROOM_DEPLOYMENT.md` |
| Terraform OCI resources | `terraform/*.tf` |
| Remote state contract | `terraform/versions.tf`, `terraform/backend.hcl.example` |
| Ansible converge | `ansible/playbooks/site.yml`, `ansible/roles/*` |
| Private secrets materialization | `ansible/playbooks/private-runtime-config.yml` |
| GitOps children | `argocd/*.yaml`, `argocd/kustomization.yaml` |
| Workloads | `apps/postgres`, `apps/mlflow`, `apps/monitoring`, `apps/argo` |
| Scratch K8s contract | `apps/scratch/**` |
| Clean-room tools | `tools/bootstrap-cloud-shell`, `tools/deploy-clean-room`, `tools/verify-clean-room` |
| CI | `.github/workflows/repository-validation.yml` |
| Unit/contract tests | `tests/unit/*.sh` |
| V1 history | `VERSION_1_BASELINE.md`, `CHANGELOG.md` `[0.1.0]` |
| V1 removal | `tests/unit/test_retired_v1_paths_contract.sh`, PR `#73` |
| Live Greenfield claim | `CHANGELOG.md` `[0.2.0]` Changed, `README.md`, runbook live acceptance |
| Stale “not yet proven” docs | `ansible/README.md` (live execution / scratch binding sentences) |
| Agent guardrails | `AGENTS.md`, `.cursor/rules/*`, `docs/AI_AGENT_WORKFLOW.md` |
| Authorship | `git log`; `LICENSE` |

Git history used: `git log --oneline`, author stats, tags `v0.1.0` /
`v0.2.0`, confirmation that `scripts/` is absent. Not used as a substitute
for reading the current tree.

Web search: not used. External product behavior (Cloud Shell auth, Helm
timeout units, OCI LaunchOptions) is taken from repository comments and
CHANGELOG live-defect writeups.

---

## Limitations and Non-Claims

Do not use this repository as evidence for:

- Production-grade, highly available, or multi-node Kubernetes.
- Managed Kubernetes (OKE) or multi-cloud.
- That GitHub Actions performed a live rebuild.
- That Prometheus, Grafana, Argo CD, or the OCI CSI provider were
  implemented from scratch.
- Vault administration, secret-value lifecycle, or KMS customer keys.
- Public HTTPS ingress, cert-manager, or internet-exposed NodePorts.
- NetworkPolicy, pod security admission, or image-digest deployment of
  general workloads.
- Workload probes, HA postgres, or non-RC MLflow images.
- Destroy automation as a first-class tool (procedure only).
- Free Tier / Always Free eligibility or zero cost.
- That `ansible/README.md` is fully consistent with `0.2.0` acceptance docs.

Implemented limits to keep visible:

- One VM, MicroK8s 1.29, hostPath scratch, postgres on boot-disk hostpath.
- Floating or RC container tags on postgres/mlflow.
- Argo Workflows `authMode: server` and namespace `admin` RoleBinding.
- Grafana/Prometheus/Argo UIs intended for SSH port-forward.
- Image selection can replace the instance on later Terraform plans.
- IAM is compartment-scoped secret-bundle read, not per-secret OCID.
- `private_runtime_config` is not inside `site.yml` (deferred activation
  inside the default converge; still run by `deploy-clean-room`).
- In-place SPC cutover document is historical/fallback.
- This analysis did not re-execute OCI apply/destroy; live-proven status is
  a repository document claim from August 2026.

### Derived Defensible Experience Statements

These are interpretations derived from the evidence above, not raw repository facts or final CV copy:

- Designed and implemented a Terraform → Ansible → Argo CD ownership split
  for a single-node OCI research cluster.
- Automated a gated clean-room deploy/verify path, including destructive-plan
  refusal, format authorization, idempotency checks, and reboot proof.
- Converted live OCI/MicroK8s/Helm/apt failures into fail-closed automation
  and static regression tests.
- Configured GitOps delivery of upstream Helm charts and authored Kubernetes
  manifests for postgres, MLflow, scratch storage, and secret consumption.
- Built static CI that validates contracts without claiming live proof.

Those statements remain bounded by the limitations above.
