<div>
  <a href="#">
    <img width="100%" src="./assets/header.svg"/>
  </a>
</div>

I build reliability-focused infrastructure and systems tooling around Linux, Kubernetes, automation, observability, and trading infrastructure. My work focuses on reproducible environments, explicit system boundaries, failure handling, and practical validation.

Recent work includes an OCI/MicroK8s platform, operational tooling for backtest and paper workflows, and Fedora networking automation.


## Engineering Focus

**Infrastructure & Reliability**  
Linux, OCI, Terraform, Ansible, Kubernetes / MicroK8s, Argo CD, Helm, Kustomize

**Operations & Observability**  
Prometheus, Grafana, reconciliation, failure drills, artifact-backed operational evidence

**Systems & Networking**  
Rust, Python, Bash, async WebSockets, NetworkManager, firewalld, WireGuard

**Data & Trading Infrastructure**  
PostgreSQL, TimescaleDB, SQLAlchemy, MLflow, market-data clients, simulation/runtime integration

**Testing & Documentation**  
pytest, integration testing, Ruff, validation tooling, Sphinx, MkDocs, ADRs, Docs-as-Code

**Development & Delivery**  
Docker, Docker Compose, Git, GitHub Actions, Dev Containers, GitOps workflows

<p>
  <img
    src="https://skillicons.dev/icons?i=linux,docker,kubernetes,terraform,ansible,rust,python,bash,postgres,prometheus,grafana,git,githubactions"
    height="36"
    title="Linux, Docker, Kubernetes, Terraform, Ansible, Rust, Python, Bash, PostgreSQL, Prometheus, Grafana, Git, GitHub Actions"
    alt="Linux, Docker, Kubernetes, Terraform, Ansible, Rust, Python, Bash, PostgreSQL, Prometheus, Grafana, Git, GitHub Actions"
  />
</p>


## Featured Engineering Work

**[TradingChassis Infrastructure](https://github.com/TradingChassis/infrastructure)**  
Single-node OCI/MicroK8s platform built around explicit ownership boundaries: Terraform provisions cloud resources, Ansible converges the host and Kubernetes bootstrap, and Argo CD owns long-lived cluster state. Includes clean-room deployment and verification, OCI Vault secret delivery, scratch storage, observability, and static validation.

**[TradingChassis Ops Lab](https://github.com/TradingChassis/tradingchassis-ops-lab)**  
Local-first operations lab for reproducible trading-system workflows, built around spec-driven runs and artifact-backed evidence. Includes reconciliation, failure drills, kill-switch gating, Prometheus/Grafana observability, and NautilusTrader-backed backtest integration, with the focus on operational behavior.

**[Fedora AirVPN Kill Switch](https://github.com/bxvtr/fedora-airvpn-killswitch)**  
Ansible and Bash automation for AirVPN WireGuard on Fedora using NetworkManager and firewalld to enforce fail-closed, leak-resistant behavior on the documented path. Includes installation/runtime tooling, verification, and live VM validation on Fedora Silverblue 43.


## Additional Engineering Work

**[OCI Secrets Store CSI Driver Provider — Oracle fork](https://github.com/TradingChassis/oci-secrets-store-csi-driver-provider)**  
Focused fork adding `linux/amd64` + `linux/arm64` container packaging, GHCR publishing, CI guardrails, and consumer integration while keeping the provider application logic upstream. The ARM64 build path was added to support the OCI Ampere environment used by TradingChassis Infrastructure.

**[Deribit Latency Tester](https://github.com/bxvtr/deribit-latency-tester)**  
Async Rust WebSocket measurement client for application-observed JSON-RPC round-trip timing, Deribit-reported processing timestamps, and private order lifecycle experiments.

**[Timescale Access](https://github.com/bxvtr/timescale-access)**  
Python SQLAlchemy/pandas wrapper around TimescaleDB/PostgreSQL with hypertable orchestration, trade-like time-series ingestion and analysis helpers, and local integration tests.

**[Deribit History Client](https://github.com/bxvtr/deribit-history-client)**  
Thin synchronous Python client for Deribit public historical endpoints with sequence-window trade retrieval and schema-snapshot tooling for response-shape experiments.


## Earlier Architecture Explorations

These repositories document an earlier custom-engine direction that was later marked demo / legacy / architectural exploration.

**[TradingChassis Core](https://github.com/TradingChassis/core)**  
Deterministic event-driven decision-engine prototype exploring canonical events, intents, policy admission, and execution-control boundaries.

**[TradingChassis Core Runtime](https://github.com/TradingChassis/core-runtime)**  
Runtime/orchestration prototype integrating hftbacktest, Core event mapping, experiment sweeps, OCI artifacts, and Argo workflow definitions.

**[TradingChassis Docs](https://github.com/TradingChassis/docs)**  
Docs-as-Code architecture corpus with canonical terminology, ADRs, concept/stack separation, and versioned MkDocs publishing.


## Open-source contributions

[**oci-prometheus-sd-proxy Helm Chart**](https://github.com/amaanx86/oci-prometheus-sd-proxy/pull/80)  
Built the initial Kubernetes Helm chart MVP with configurable deployment, external secret integration, security defaults, health probes, and usage docs; foundation later evolved into the [production-ready Helm chart](https://github.com/amaanx86/oci-prometheus-sd-proxy/pull/88).

[**View all external pull requests**](https://github.com/search?q=is%3Apr+author%3Abxvtr+-user%3Abxvtr+-user%3ATradingChassis+is%3Amerged&type=pullrequests)


## How I Work

I prefer small, reviewable changes, explicit ownership boundaries, and validation that makes technical claims traceable to code or operational evidence. I favor security-minded defaults and treat static CI separately from live validation. I use AI tools as part of the development workflow while keeping implementation, diffs, tests, and verification human-reviewed.

Deeper technical notes, design decisions, limitations, and implementation evidence, are available in the [portfolio records](https://github.com/bxvtr/bxvtr/tree/main/portfolio).


<div>
  <a href="#">
    <img width="100%" src="./assets/footer.svg"/>
  </a>
</div>
