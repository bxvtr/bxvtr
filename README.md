<div>
  <a href="#">
    <img width="100%" src="./assets/header.svg"/>
  </a>
</div>

I build reliability-focused infrastructure and systems tooling around Linux, Kubernetes, automation, and observability. My work focuses on reproducible environments, explicit system boundaries, failure handling, and practical validation.

Recent work includes an OCI/MicroK8s platform, operational tooling for backtest and synthetic paper workflows, and Fedora networking automation.


## Featured Engineering Work

**[TradingChassis Infrastructure](https://github.com/TradingChassis/infrastructure)**  
Reproducible single-node OCI/MicroK8s platform built around explicit ownership boundaries: Terraform provisions cloud resources, Ansible converges Linux host configuration and Kubernetes bootstrap, and Argo CD owns long-lived cluster state. Includes a live-validated clean-room deployment path, reboot/post-reboot convergence, OCI Vault-backed secret delivery, observability, and lifecycle validation.

**[TradingChassis Ops Lab](https://github.com/TradingChassis/tradingchassis-ops-lab)**  
Local-first operations and reliability lab built around reproducible, spec-driven workflows and artifact-backed operational evidence. Includes reconciliation checks, failure drills and runbooks, file-based safety controls, Prometheus/Grafana observability, and NautilusTrader-backed workflow integration focused on operational behavior.

**[Fedora AirVPN Kill Switch](https://github.com/bxvtr/fedora-airvpn-killswitch)**  
Ansible and Bash automation for AirVPN WireGuard on Fedora using NetworkManager and firewalld, designed around fail-closed, leak-resistant behavior on the documented path. Includes installation/runtime tooling, layered validation, installed-state verification, and opt-in live lifecycle testing in disposable Fedora virtual machines.


## Additional Engineering Work

**[OCI Secrets Store CSI Driver Provider — Oracle Fork](https://github.com/TradingChassis/oci-secrets-store-csi-driver-provider)**  
Focused upstream fork adding `linux/amd64` + `linux/arm64` container packaging, GHCR publishing, CI guardrails, and consumer integration while keeping the provider application logic upstream. The ARM64 build path was added to support the OCI Ampere environment used by TradingChassis Infrastructure.

**[Timescale Access](https://github.com/bxvtr/timescale-access)**  
Python tooling around PostgreSQL/TimescaleDB for time-series ingestion, hypertable management, querying, and local integration testing.


## Earlier Architecture Explorations

These repositories document an earlier custom-engine direction that was later marked legacy / architectural exploration.

**[TradingChassis Core](https://github.com/TradingChassis/core)**  
Deterministic event-driven decision-engine prototype exploring canonical events, intents, policy admission, execution control, and explicit runtime boundaries.

**[TradingChassis Core Runtime](https://github.com/TradingChassis/core-runtime)**  
Runtime and orchestration prototype around Core with local backtesting, Argo/Kubernetes workflows, OCI artifact handling, and self-hosted container build workflows using cached Kaniko builds.


## Open-Source Contributions

[**oci-prometheus-sd-proxy — Helm Chart**](https://github.com/amaanx86/oci-prometheus-sd-proxy/pull/80)  
Contributed the initial Kubernetes Helm chart MVP with configurable deployment, external secret integration, security defaults, health probes, validation, and usage documentation; contribution merged upstream and later evolved into the [production-ready Helm chart](https://github.com/amaanx86/oci-prometheus-sd-proxy/pull/88).

[**View all external merged pull requests**](https://github.com/search?q=is%3Apr+author%3Abxvtr+-user%3Abxvtr+-user%3ATradingChassis+is%3Amerged&type=pullrequests)


## How I Work

I prefer small, reviewable changes, explicit ownership boundaries, and validation that makes technical claims traceable to code or operational evidence. I favor security-minded defaults and treat static validation separately from live validation.

I use AI-assisted development extensively and review diffs, tests, system behavior, and implementation decisions before treating changes as complete.

Deeper technical notes, design decisions, limitations, and implementation evidence are available in the [portfolio records](https://github.com/bxvtr/bxvtr/tree/main/portfolio).


<div>
  <a href="#">
    <img width="100%" src="./assets/footer.svg"/>
  </a>
</div>