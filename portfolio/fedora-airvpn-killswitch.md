# Fedora AirVPN Kill Switch

Evidence-based engineering and learning record for
[`bxvtr/fedora-airvpn-killswitch`](https://github.com/bxvtr/fedora-airvpn-killswitch).

This file is the canonical portfolio analysis record for this project. The
repository remains the technical source of truth for implementation details.
This file is not a CV, README rewrite, or marketing summary.

Repository analyzed: local clone at tag **`v0.1.0`** / `main`
(`2d1a5d9`, 2026-08-04). License: MIT, copyright `bxvtr` /
`bxvtrdev@protonmail.com`. Git authorship on `main`: **43 commits by `bxvtr`**.
Three unmerged Dependabot commits exist on remote branches and are **not**
part of `v0.1.0`.

How to read status labels:

| Label | Meaning in this file |
| --- | --- |
| Implemented | Present in Ansible, Bash, firewalld/NM commands, or tests |
| Live validated | Observed in the documented disposable-VM run |
| Statically validated only | Unit tests, lint, syntax, or code review — no live traffic |
| Implemented, not live validated | Code exists; the documented live evidence does not cover it |
| Intentional | Deliberate design choice, not a missing feature |
| Planned / deferred | Roadmap only |
| Configured but unused | Pinned or installed, not exercised by this tree |
| Inferred | Reasonable conclusion from multiple artifacts |
| Uncertain | Insufficient evidence |

Source-of-truth rule: implementation wins over documentation.
**Implementation and live validation are not the same.**

This project is **unofficial**. It is not affiliated with AirVPN. It does not
download AirVPN configs, authenticate against AirVPN, or manage AirVPN
accounts. It consumes **user-exported WireGuard `.conf` files**.

Security bound that must not be inflated:

> Fail-closed and leak-**resistant** on a documented path.
> Not guaranteed leak-proof, not formally verified, not a universal
> production security product.

---

## Project Overview

**Status:** Implemented (`v0.1.0`). Live validated on one Atomic VM path.

The repository is Fedora-only Ansible + Bash tooling that:

1. Imports user-supplied AirVPN WireGuard configs into **NetworkManager**.
2. Places managed VPN interfaces in firewalld zone `airvpn` and selected
   physical interfaces in `vpn-underlay`.
3. Installs **firewalld policy objects** so host-originated traffic to the
   underlay is **REJECT** except numeric WireGuard endpoint UDP (plus DHCP/NDP
   allowances), while traffic to the VPN zone is **ACCEPT**.
4. Leaves imported tunnels **inactive** until an explicit `airvpn-switch`.
5. Provides runtime commands, installed-state checks, an opt-in live VM
   lifecycle test, and an uninstall playbook that aims to remove
   **project-owned** state only.

What it is:

- A **local-host** NetworkManager + firewalld kill-switch lab for AirVPN
  WireGuard on Fedora (package-based and Atomic).
- A **fail-closed** design: ordinary Internet via managed physical
  interfaces is blocked unless a managed tunnel is up, or the operator
  explicitly disables the project policies / uninstalls.
- A project that separates **static validation**, **Ansible check mode**,
  **live VM integration**, and **installed-state verification**.

What it is not:

- An AirVPN client, API integration, or credential manager.
- A guaranteed leak-proof VPN product.
- Multi-distro or remote-controller Ansible (v0.1.0 is
  `controller == managed host`).
- A general-purpose firewall manager.
- Live-proven on physical Wi-Fi, suspend/resume, dual uplink, or
  package-based Workstation (those remain target-family, not proven).

Evidence:

```text
- README.md
- docs/KNOWN_LIMITATIONS.md
- docs/RELEASE_NOTES_v0.1.0.md
- SECURITY.md
- playbooks/install.yml
- roles/airvpn_client/
```

---

## What I Built

Attribution is strong: a two-day single-author sequence (`b98f69a` 2026-08-02
through merge `2d1a5d9` 2026-08-04), then a signed tag `v0.1.0`. This provides
strong evidence that I was the primary author and maintainer of the repository's
Ansible, Bash tooling, tests, and documentation. Live validation is narrower:
only the scenarios explicitly covered by the documented Fedora Silverblue 43
VM run should be described as live-validated. This does not mean NetworkManager,
firewalld, WireGuard, or Ansible were written here.

### Authored in this repo — Implemented

1. **Controller bootstrap** — user-run `bootstrap.sh`: repo-local venv,
   pinned `ansible-core`/lint tools, pinned Galaxy collections, then local
   install playbook with `--ask-become-pass`. Refuses root. Refuses config
   sources inside the clone.
2. **Ansible role `airvpn_client`** — validate → detect → dependencies →
   scripts → firewall → NetworkManager → offline verify.
3. **Fedora Atomic path** — detect `/run/ostree-booted`; **no DNF**; fail
   with `rpm-ostree install` instructions by default; optional layering
   still **does not reboot**.
4. **firewalld zones + HOST→zone policies** via `firewall-cmd` (no
   ansible.posix policy module in 2.2.2).
5. **Runtime Bash commands** under `/usr/local/libexec/airvpn-client` with
   `/usr/local/bin` symlinks: import, switch, firewall-sync, protect,
   status, check, killswitch.
6. **Shared library** `airvpn-common.sh` — lock, NM terse parsing, numeric
   endpoint parse, rich-rule ownership, handshake wait, `ip route get`
   effective-route checks, redaction.
7. **Opt-in live VM runner** `tools/integration-test-vm` with explicit
   consent, virt detection, snapshot confirmation, leak probes, uninstall.
8. **Uninstall audit snapshot** collector (read-only; atomic publish).
9. **Static validation** `tools/validate-safe` + Bash unit tests + CI
   (lint/syntax/unit/gitleaks) that **must not** run live networking.
10. **Cursor/agent guardrails** (instructions + allowlists; **not** a sandbox).

### Dependency-provided (not authored here)

- NetworkManager / `nmcli`, firewalld / `firewall-cmd`, WireGuard kernel +
  `wg`, systemd.
- ansible-core, ansible.posix (`firewalld` zone module), collections.
- Gitleaks, ShellCheck, yamllint, ansible-lint, pre-commit.

### Configured but unused

- `pytest==9.1.1` is pinned in `requirements-controller.txt` and listed in
  the README. **No pytest tests exist**; the suite is Bash under
  `tests/unit/`.
- `community.general` is pinned and installed; playbooks talk to NM via
  `nmcli` (`ansible.builtin.command`), not `community.general.nmcli`.

Evidence:

```text
- git log / tag v0.1.0
- bootstrap.sh
- roles/airvpn_client/**
- tools/integration-test-vm
- tools/validate-safe
- tests/unit/
- requirements-controller.txt
- requirements.yml
```

---

## Architecture and Ownership Boundaries

**Status:** Implemented.

### Layers

| Layer | Owner | Responsibility |
| --- | --- | --- |
| Controller bootstrap | `bootstrap.sh` (normal user) | venv, pins, Galaxy, invoke playbook |
| Install / uninstall | Ansible playbooks + role | Detect OS, packages, deploy scripts, baseline firewall, NM assign, verify, cleanup |
| Runtime | Bash helpers | Import, switch, sync, protect, status, check, killswitch |
| Host networking | NetworkManager + firewalld + WireGuard | Actual packets, zones, policies, tunnels |
| Live proof | `tools/integration-test-vm` | Opt-in disposable VM lifecycle |
| Static proof | `tools/validate-safe`, unit tests, CI | Syntax/lint/mocks/secrets — not traffic |

Task order is contract-tested: **firewall before NetworkManager**, so
`airvpn-import` → `airvpn-firewall-sync` has policies to mutate.

```text
Evidence:
- roles/airvpn_client/tasks/main.yml
- tests/unit/test_structure.sh
```

### Project-owned vs not owned

**Project-owned (install creates / uninstall tries to remove):**

- firewalld zones `airvpn`, `vpn-underlay` (configurable names)
- policies `airvpn-host-vpn`, `airvpn-host-under` plus hardcoded legacy
  names `airvpn-host-to-vpn`, `airvpn-host-to-underlay`
- NetworkManager profiles whose **name** starts with `AirVPN - `
- copied configs under `/etc/airvpn-client/configs` (kept by default)
- scripts under `/usr/local/libexec/airvpn-client` and `/usr/local/bin`
  wrappers
- `/etc/airvpn-client/airvpn-client.conf`
- `/var/lib/airvpn-client` (owned-rules state, pending-reload marker)
- `/run/airvpn-client.lock` (runtime; cleared by reboot)

**Not project-owned:**

- AirVPN account, source `.conf` directory (must live **outside** the clone)
- firewalld **service** globally (enabled at install; not stopped on
  uninstall or by `airvpn-killswitch disable`)
- Unrelated NM profiles
- OS image / rpm-ostree layers / DNF packages
- Administrator rich rules on the underlay policy that are **not**
  project-shaped endpoint rules
- Controller `.venv` / `.ansible` (repo-local; uninstall does not delete)

**Ownership gap (statically identified):** install/uninstall operate on
**names**, not a creation ledger. A pre-existing zone/policy with the same
name can be reconfigured or deleted. No live test of that collision.

```text
Evidence:
- playbooks/uninstall.yml
- roles/airvpn_client/files/airvpn-firewall-sync
- roles/airvpn_client/files/airvpn-killswitch
- docs/KNOWN_LIMITATIONS.md (Pre-existing firewalld Objects)
```

---

## NetworkManager Integration

**Status:** Implemented. Live validated for **virtual Ethernet** on
Silverblue 43. Wi-Fi types are implemented; **physical Wi-Fi not live
validated**. Post-install new profiles: **implemented helper, not
automatic, not live validated**.

### Physical profiles

`airvpn-select-physical-uuids.sh` lists UUIDs of type `802-11-wireless`
and/or `802-3-ethernet`, with `--include` / `--exclude`. Install:

1. Sets `connection.zone` to `vpn-underlay` when it differs.
2. Calls `airvpn_ensure_runtime_zone`: if the connection is **active**,
   `nmcli device reapply` and require firewalld runtime zone match.
3. **Fails the play** if any active interface cannot bind (assert over
   collected loop results so a later success cannot hide an earlier
   failure).

Inactive connections: profile metadata only (runtime bind skipped).

`airvpn-protect-connection` does the same for one name/UUID after install.
It **refuses** WireGuard profiles (will not move a tunnel into underlay).
Only Wi-Fi and Ethernet are accepted.

**Gap (documented, keep):** profiles created **after** install are not
auto-assigned. No NetworkManager dispatcher ships in v0.1.0 (roadmap
v0.2.0).

### Managed VPN profiles

Identified by name prefix `AirVPN - ` (not a secret NM flag). Import
hardens:

| Property | Value | Intent |
| --- | --- | --- |
| `connection.autoconnect` | `no` | No tunnel at boot; fail-closed until `airvpn-switch` |
| `connection.zone` | `airvpn` | HOST→vpn policy ACCEPT |
| `connection.interface-name` | `avpn` + 11 hex of SHA-256(pubkey\|endpoint) | IFNAMSIZ-safe (≤15) |
| `ipv4/ipv6.dns-priority` | `-100` | VPN DNS wins when the profile is up |
| `ipv4/ipv6.dns-search` | `~.` | Routing domain: this DNS for all names |
| `ipv4/ipv6.never-default` | `no` | Full-tunnel default via WG |
| `ipv6.method` | `disabled` if `AIRVPN_ENABLE_IPV6` is false | Optional IPv6 off |

NetworkManager **may activate on import**. `airvpn-import` then
`airvpn_ensure_connection_inactive` (timeout 45s). Failure: **leave the
profile**, abort further imports (not a full rollback of earlier
successful imports).

Install requires **zero** active managed tunnels before offline verify.

Idempotency: `--mode add` refreshes the root-owned `.conf` copy and
re-hardens NM metadata; it does **not** re-import keys/peers. `--mode
replace` deletes managed profiles and managed copies, then re-imports.

```text
Evidence:
- roles/airvpn_client/tasks/networkmanager.yml
- roles/airvpn_client/files/airvpn-import
- roles/airvpn_client/files/lib/airvpn-select-physical-uuids.sh
- roles/airvpn_client/files/airvpn-protect-connection
- roles/airvpn_client/files/lib/airvpn-common.sh (airvpn_ensure_runtime_zone)
```

---

## firewalld and Kill-Switch Design

**Status:** Implemented. Live validated (Silverblue 43 / vEth) for
fail-closed without VPN and for policy presence after install.

### Traffic contract (implementation)

Not a parallel nftables/iptables ruleset. Two **policy objects**:

```text
ingress HOST → egress zone airvpn          target ACCEPT
  policy name default: airvpn-host-vpn

ingress HOST → egress zone vpn-underlay    target REJECT
  policy name default: airvpn-host-under
  + UDP accept to each imported numeric endpoint:port
  + ipv4 dhcp accept
  + ipv6 dhcpv6-client, neighbour-solicitation/advertisement,
    router-advertisement accept
```

Physical devices in `vpn-underlay`; WireGuard devices in `airvpn`.
Host-originated “ordinary Internet” to the underlay is rejected. Tunnel
traffic to the VPN zone is accepted. Endpoint UDP is the **bootstrap**
path so a handshake can occur while the underlay is otherwise closed.

`ansible.posix.firewalld` creates/deletes **zones** with `permanent: true`
only (`immediate: true` is invalid for zone ops in 2.2.2). Runtime
activation is an explicit `firewall-cmd --reload` after
`firewall-cmd --check-config`.

Policy names: charset `^[A-Za-z0-9][A-Za-z0-9_-]*$`, max **18** characters
(firewalld/iptables chain limit). Validated **before** any firewall
mutation. Defaults were shortened from historical
`airvpn-host-to-vpn` / `airvpn-host-to-underlay` (the latter may never
have been creatable at 18 chars). Legacy delete runs **after** current
policies exist, so one reload does not open a window with neither policy.
Historical names are **YAML literals**, not extra-vars, so operators
cannot widen the delete set.

### Endpoint exception ownership

`airvpn-firewall-sync`:

- Builds desired rich rules only from **numeric** endpoints in managed
  copies.
- Tracks project rules in
  `/var/lib/airvpn-client/owned-underlay-endpoint-rules`.
- Removes the union of owned-file ∩ live **project-shaped** rules that
  are no longer desired. **Does not** delete unrelated admin UDP rules.
- Validates rule text before `firewall-cmd`.
- Failed add/remove **aborts before rewriting** the owned-rules file.
- Pending-reload marker forces retry if `--reload` failed after permanent
  changes.

**Intentional residual surface:** **all** imported endpoints stay allowed
on the underlay, not “current server only” (roadmap item). That is UDP to
those IPs/ports even when that tunnel is down.

### `airvpn-killswitch`

Operates **only** on the two project policy targets:

- `enable` → VPN ACCEPT, underlay REJECT, check-config, reload.
- `disable` → underlay ACCEPT after typing `DISABLE KILLSWITCH` (or
  `--force`). Does **not** stop firewalld.
- `status` → print targets / MISSING.

This is an explicit escape hatch, not a silent “make the network work”.

```text
Evidence:
- roles/airvpn_client/tasks/firewall.yml
- roles/airvpn_client/tasks/validate.yml
- roles/airvpn_client/files/airvpn-firewall-sync
- roles/airvpn_client/files/airvpn-killswitch
- docs/INTEGRATION_TESTING.md (policy name limits)
```

---

## WireGuard Endpoint Handling

**Status:** Implemented. Statically tested. Live import used real AirVPN
numeric endpoints on the Silverblue path.

Parser (`airvpn_parse_endpoint`):

| Input | Result |
| --- | --- |
| `192.0.2.10:1637` | IPv4, rc 0 |
| `[2001:db8::10]:1637` | IPv6, rc 0 |
| hostname:port | **rc 2** — rejected |
| unbracketed IPv6 | rejected |
| missing/invalid port, bad octets | rc 1 |

**Security design:** a hostname endpoint would require DNS **before** the
tunnel exists. With underlay REJECT, that DNS either fails (cannot
connect) or would need an underlay DNS exception (leak/bootstrap hole).
v0.1.0 refuses hostnames and documents “re-export numeric IPs from
AirVPN.”

Private keys: `airvpn_wg_field` extracts values for validation; status and
logs are written not to print them. `airvpn_redact` strips
`PrivateKey` / `PresharedKey` lines in `resolvectl` dumps. Import copies
via `install -m 0600 -o root` into `/etc/airvpn-client/configs` and a
`mktemp` dir named `<ifname>.conf` (NM import requires basename =
interface + `.conf`, IFNAMSIZ-1 = 15). Temp dir removed on success and
failure (`trap RETURN`).

Fixtures use documentation-range IPs and obvious fake keys (`A{…}EE=`),
allowlisted in `.gitleaks.toml`.

```text
Evidence:
- roles/airvpn_client/files/lib/airvpn-common.sh (parse, iface, wg_field, redact)
- roles/airvpn_client/files/airvpn-import
- tests/unit/test_parsing.sh
- tests/fixtures/*.conf
- .gitleaks.toml
```

---

## DNS, Routing, IPv4 and IPv6

**Status:** Configuration implemented. Live validated on Silverblue 43 /
vEth for: fail-closed IPv4/IPv6/DNS **without** VPN; with VPN: handshake,
effective IPv4 (+ IPv6 where the profile supports it) policy routing via
WG iface, DNS via VPN profile, public IPv4 via AirVPN.

### DNS leak resistance (config vs proof)

**Intended config:** VPN `dns-priority -100` and `dns-search ~.` so, when
the tunnel is up, systemd-resolved should use VPN DNS for all queries.
Underlay ordinary DNS is blocked by HOST→underlay REJECT (except whatever
DHCP/NDP need).

**`airvpn-check --offline`** asserts those NM properties. It does **not**
send DNS queries.

**Live VM `phase_offline`:** probes public HTTP v4/v6 and `getent` DNS.
HTTP success **fails** the phase. DNS success while HTTP is blocked is
**WARN** (cache possible), not automatic FAIL.

README: a public DNS-leak **website** alone is not adequate verification.

### Routing

NetworkManager WireGuard often puts the default route in a **policy
routing table** (`ip4-auto-default-route`). Checking only `ip route show
default` on table main is a **false negative**. `airvpn-check --online`
uses `ip route get` to RFC 5737 `192.0.2.1` and RFC 3849 `2001:db8::1`
and requires the selected **dev** to be the managed WG interface.

IPv6 online: if the effective route is not via the tunnel, check
**warns** (IPv4-primary AirVPN profiles) rather than failing; underlay
should still REJECT.

### IPv6 elsewhere

- Endpoint exceptions: family ipv6 when endpoint is bracketed.
- Baseline NDP/DHCPv6-client rich rules on underlay.
- `airvpn_enable_ipv6: false` disables `ipv6.method` on managed profiles.

Do **not** claim “IPv6 secure on all Fedora topologies.” Live IPv6
fail-closed without VPN was observed on the documented VM; physical
Wi-Fi/roaming IPv6 is **not live validated**.

```text
Evidence:
- roles/airvpn_client/files/airvpn-check (check_online_tunnel)
- roles/airvpn_client/files/lib/airvpn-common.sh (airvpn_effective_route_uses_iface)
- tests/unit/test_routing_verify.sh
- tools/integration-test-vm (phase_offline, phase_first_vpn)
- docs/INTEGRATION_TESTING.md (Online routing verification)
```

---

## Profile Import and Switching

**Status:** Implemented. Live validated: two profiles imported inactive;
activate exactly one; switch with concurrent public-IP probes; disconnect
returns to blocked underlay.

### Import (`airvpn-import`)

Order: validate source dir (not world-readable; `[Interface]`/`[Peer]`;
PrivateKey present but not logged; Endpoint numeric) → optional replace
wipe → per-file copy + NM import via temp ifname → harden → force
inactive → `airvpn-firewall-sync`.

`--dry-run` prints actions; does not mutate.

**Not a database transaction:** a mid-batch failure can leave earlier
profiles imported. Replace mode deletes all managed profiles first
(window with no VPN; kill switch should still REJECT underlay). Exclusive
`flock` on `/run/airvpn-client.lock` (FD 9; nested import→sync allowed;
`AIRVPN_LOCK_HELD` env is **ignored**).

### Switch (`airvpn-switch`)

`activate_managed_uuid`:

1. Disconnect **all** managed tunnels; wait inactive (45s) or die.
2. `nmcli connection up`; wait active (60s) or die.
3. Wait handshake ≤180s age, poll 60s; on failure **bring the new tunnel
   down** and die. Message: kill switch remains active.
4. Require **exactly one** active managed VPN.

During step 1–2 there is **no** tunnel. Ordinary Internet stays blocked
**if** underlay policy is still REJECT and physical ifaces stay in
`vpn-underlay`. That is the “blocked during server switches” claim.

**Live evidence:** `phase_switch` backgrounds IPv4 public-IP lookups,
classifies `baseline_leak` vs VPN IPs, fails if baseline appears.
Requires two managed profiles unless `--skip-switch-test`.

`--disconnect`: tear down managed tunnels; **does not** disable kill
switch. Live: `phase_forced_disconnect` asserts public HTTP v4/v6 fail
after disconnect.

Interactive menu if no flags. `--uuid` / `--name` / `--disconnect`
mutually exclusive. Ambiguous names die (use UUID).

```text
Evidence:
- roles/airvpn_client/files/airvpn-switch
- roles/airvpn_client/files/airvpn-import
- tools/integration-test-vm (phase_switch, phase_forced_disconnect)
- tests/integration/lib/network-probes.sh
```

---

## Runtime Safety and Verification

**Status:** Implemented. Offline/online checks are **installed-state**
tools, not pre-install proofs.

### Fail-closed mechanisms (concrete)

| Trigger | Mutation / halt | Fail-open window intended? | Proof |
| --- | --- | --- | --- |
| Install finishes | Profiles imported then forced inactive; underlay REJECT | No ordinary Internet until `airvpn-switch` | Live (inactive after import; offline probes) |
| No VPN / after disconnect | Underlay stays REJECT | No | Live HTTP block |
| Switch | Disconnect all, then up + handshake | Tunnel-less interval; underlay still REJECT | Live IP probes during switch |
| Activation fail | Die after failed `nmcli up` | No | Code + unit/integration orchestration; live if it occurred |
| Handshake fail | Down the new tunnel, die | No | Code; live runner would FAIL first-vpn |
| Runtime zone bind fail at install | Play assert fail | Prefer block vs continue | Code; live on vEth success path |
| Hostname endpoint | Die before firewall sync of that file | No DNS-underlay exception | Unit tests |
| Invalid policy name | Validate tasks fail first | No firewall mutation | Ansible validate.yml |
| `killswitch disable` | Underlay ACCEPT | **Yes, explicit** | Confirmation phrase |
| Uninstall | Delete policies/zones | **Yes, by design** | Live connectivity returned |

There is **no** automatic “repair connectivity” path. Recovery after a
failed switch is: inspect `airvpn-status` / `airvpn-check`; underlay
should still block; restore VM snapshot if the live test left uncertain
state. The runner **does not flush nftables** on failure.

### `airvpn-check`

Exit 0 all pass, 1 failures, 2 usage. Does not call `airvpn_require_root`
(privileged reads may still fail as non-root).

**`--offline`:** commands, NM/firewalld active, zones/policies present,
underlay target REJECT/DROP, vpn ACCEPT, endpoint exceptions match
managed configs (stale project rules fail even if expected set is empty),
managed profile autoconnect/zone/DNS/`~.`, active physical Wi-Fi/Ethernet
profile **and runtime** zone, managed VPN count (fail if >1; 0 is OK).

**`--online`:** additionally exactly one active managed VPN, handshake
≤180s, effective IPv4 (and IPv6 if enabled) `ip route get` via WG iface.

### `airvpn-status`

firewalld/policies/zones, physical connections (warn if active and not
underlay), managed VPNs, handshake age, `wg show endpoints` (not keys),
count, `resolvectl status` piped through `airvpn_redact`.

```text
Evidence:
- roles/airvpn_client/files/airvpn-check
- roles/airvpn_client/files/airvpn-status
- roles/airvpn_client/tasks/verify.yml
- playbooks/verify.yml
```

---

## Fedora Atomic Support

**Status:** Implemented. **Live validated on Fedora Silverblue 43**
(Atomic Desktop) — the documented live environment.

Detection: `/run/ostree-booted` (bootstrap + Ansible `stat`).

Package-based Fedora: `ansible.builtin.package` for NM, firewalld,
wireguard-tools, python3, python3-firewall.

Atomic:

- **Does not** use DNF to mutate the host image.
- Default: **fail** with exact `rpm-ostree install …` + reboot + re-run
  bootstrap. `python3-firewall` is checked via `package_facts` (no CLI).
- Optional `airvpn_atomic_layer_missing_packages: true`: `rpm-ostree
  install --idempotent`, then **fail** with “reboot yourself; this role
  will not reboot.”
- Persistent files only under `/etc`, `/var`, `/usr/local` (compatible
  with ostree).

**Inference:** avoiding an unattended reboot is a safety choice on a
machine whose network is about to be fail-closed.

Uninstall does **not** unlayer rpm-ostree packages (intentional;
prerequisites stay). Live uninstall snapshots showed package/ostree state
matching baseline because the test host already had the tools.

```text
Evidence:
- bootstrap.sh (ATOMIC detection)
- roles/airvpn_client/tasks/detect.yml
- roles/airvpn_client/tasks/dependencies_fedora_atomic.yml
- docs/KNOWN_LIMITATIONS.md
```

---

## Controller Bootstrap and Ansible

**Status:** Implemented.

Why Ansible cannot install itself: the playbook is the thing that needs
Ansible. Bootstrap is a **user-owned** venv under the clone, not a
system-wide Ansible.

Constraints:

- Fedora `ID` from `/etc/os-release` only.
- Python ≥ 3.12 (pinned ansible-core 2.21.x).
- Not EUID 0 (prevents root-owned `.venv` / `.ansible`).
- `--config-source` absolute and **outside** `ROOT_DIR`.
- `--skip-playbook`: controller only, no become prompt.
- `--check`: `--check --diff` still prompts become because check mode
  inspects privileged state. Documented as **not** a leak-prevention
  proof. Some NM/firewalld ops are not fully simulated.
- Playbooks: `hosts: localhost`, `connection: local`, `become: true`.
- Pins: `ansible-core==2.21.2`, `ansible-lint==26.6.0`,
  `yamllint==1.38.0`, `pre-commit==4.6.1`, collections posix 2.2.2 /
  general 13.2.0 (verified 2026-08-02). Dependabot bumps exist unmerged.

Install enables and starts NetworkManager and firewalld. That global
enablement is **not** reverted on uninstall.

```text
Evidence:
- bootstrap.sh
- ansible.cfg
- playbooks/install.yml
- inventory/localhost.yml
- tests/unit/test_bootstrap_become.sh
```

---

## Static Validation vs Live Integration Testing

**Status:** Both implemented; **different proof levels**. CI is static
only.

### Static (`tools/validate-safe`)

Refuses root. `bash -n`, ShellCheck, shfmt `-d` (never `-w`), yamllint,
ansible-lint, playbook `--syntax-check` with dummy config path, unit
suite, optional gitleaks, `git diff --check`, forbidden tracked paths,
no AirVPN `.conf` outside fixtures. **Explicitly must not invoke**
`integration-test-vm`. Omits pre-commit because hooks may download and
shfmt `-w` writes.

Unit tests (mocked / no live NM): parsing, lock, firewall ownership,
import filenames, routing verify, structure (task order, Atomic, add
mode), bootstrap become argv, agent-safety files, integration
orchestrator helpers, repeatable uninstall source path, audit snapshot
SIGINT/atomic publish.

### Ansible check mode

Best-effort planning + privileged inspection. Not live enforcement.

### Live VM (`tools/integration-test-vm`)

Opt-in. Requires `--i-understand-this-modifies-networking`. Detects
virtualization (`systemd-detect-virt`); `--allow-non-vm` is discouraged.
Asks for snapshot confirmation. Human become password per Ansible
process; runner does not store it. Default **uninstalls** on success.
Failed runs can leave uncertain state; **restore snapshot**, do not
flush nftables.

Phases: preflight → static → syntax/check-mode → baseline IP → install →
offline fail-closed probes → first VPN → redacted diagnostics →
server-switch leak probes → forced disconnect → optional suspend
(**off by default**) → idempotent reinstall → uninstall → summary.

### What the repo says was live validated

**Fedora Silverblue 43, KVM/Fedora Boxes, virtual Ethernet.**

Includes: install; two WG profiles inactive; fail-closed IPv4/IPv6/DNS
without VPN; one active VPN; handshake; policy routing; DNS via VPN;
public IPv4 via AirVPN; uninstall + reboot snapshot comparison;
repeatable second uninstall after repo-sourced `airvpn-common.sh`.

**Not live validated (keep this list):** physical Wi-Fi; physical
Ethernet hardware; post-install new NM profiles; suspend/resume
(optional in runner, off); roaming, airplane, dual uplink, tethering,
captive portal; package-based Workstation; concurrent uninstall vs
runtime lock; pre-existing same-named firewalld objects; every Fedora
edition.

```text
Evidence:
- tools/validate-safe
- tools/integration-test-vm
- docs/INTEGRATION_TESTING.md
- docs/KNOWN_LIMITATIONS.md
- tests/unit/run_all.sh
- .github/workflows/lint.yml (guards against live runner)
```

---

## Security and Secret Handling

**Status:** Implemented as **careful file handling + scanning**, not
secret-management automation (no vault, no AirVPN API).

- Source configs: operator-supplied, outside Git; bootstrap refuses
  in-repo paths; world-readable source **dir** is fatal on import.
- Managed copies: root, `0700` dir / `0600` files.
- `.gitignore`: `*.conf` except fixtures; `config.yml`; keys, `.env`.
- Gitleaks: full-history CI; extra PrivateKey regex; fixture allowlist.
- Status/diagnostics: redaction; `wg show endpoints` not private keys.
- Uninstall default **keeps** managed copies (private keys remain unless
  `airvpn_uninstall_delete_configs=true`).
- Cursor allowlist: only git inspect + `validate-safe` /
  `check-agent-safety`. Docs state this is **not** a sandbox.

```text
Evidence:
- .gitignore
- .gitleaks.toml
- .github/workflows/secret-scan.yml
- SECURITY.md
- .cursor/permissions.json
- .cursor/rules/safe-local-development.mdc
- tools/check-agent-safety
```

---

## Uninstall and State Ownership

**Status:** Implemented. Live validated (Silverblue 43) including
repeatable second run and after-reboot snapshots.

Requires `-e airvpn_uninstall_confirmed=true`.

Removes: managed NM profiles (prefix match), underlay physical profiles’
zone → **`airvpn_restore_zone` default `public`** (not original zone),
current + legacy policy names, project zones, scripts/wrappers, libexec
tree, `/var/lib/airvpn-client`. Then `--check-config` and reload. Verifies
policies **absent** (`--info-policy` success is failure).

**Live confirmed:** pre-install Ethernet had empty `connection.zone`
(default `FedoraWorkstation`); after uninstall it was explicit `public`.
Documented as restore-zone behavior, not unexplained damage.

**Does not:** take `/run/airvpn-client.lock` (race with runtime commands;
statically identified); delete controller venv; unlayer packages; disable
firewalld; delete configs by default.

Repeatable uninstall: sources `roles/.../airvpn-common.sh` from the
**checkout**, because the first uninstall deletes libexec. That was a
live-discovered defect, then fixed and re-validated.

Audit tool: read-only phased snapshots under `/var/tmp/...`; incomplete
captures must not publish as success (SIGINT/atomic publish hardening in
the 0.1.0 window).

```text
Evidence:
- playbooks/uninstall.yml
- tests/unit/test_uninstall_repeatable.sh
- tests/unit/test_uninstall_audit_snapshot.sh
- tools/uninstall-audit-snapshot
- docs/KNOWN_LIMITATIONS.md (Uninstall Behavior)
```

---

## Testing and CI

**Status:** Implemented. CI = Ubuntu 24.04 static jobs only.

| Job | What |
| --- | --- |
| lint.yml | yamllint 1.38.0, ansible-lint via controller reqs, ShellCheck, unit tests; **grep-guards** that workflows and `validate-safe` never run `integration-test-vm` or `uninstall-audit-snapshot` |
| syntax.yml | `ansible-playbook --syntax-check`, `bash -n` |
| secret-scan.yml | Gitleaks 8.30.1, `fetch-depth: 0`, `--redact` |

Pinned GitHub Actions SHAs. `permissions: contents: read`. Dependabot
weekly for pip + Actions.

pre-commit: trailing whitespace, YAML, detect-private-key, yamllint,
ansible-lint, shellcheck, shfmt `-w`, gitleaks.

**CI does not** run NetworkManager, firewalld, WireGuard, or install
playbooks against a host.

```text
Evidence:
- .github/workflows/*.yml
- .pre-commit-config.yaml
- tests/unit/*.sh
```

---

## Reliability and Failure Handling

Patterns actually present:

- **Validate before mutate** — policy/zone names, config source, endpoints,
  rich-rule text, `firewall-cmd --check-config` before reload.
- **Fail closed rather than auto-repair** — handshake failure downs the
  new tunnel; runtime zone failure fails install; no silent kill-switch
  disable.
- **Idempotency where cheap** — `ALREADY_ENABLED` on policy ingress,
  absent legacy policies OK, firewall-sync no-op when in sync, add-mode
  import skip of existing ifname.
- **Partial-failure honesty** — import is not transactional; uninstall
  partial recovery beyond the repeatable-libexec fix is **not** fully
  live validated; live runner prefers snapshot restore.
- **Serialization** — flock for mutating `airvpn-*`; uninstall **outside**
  that lock (known gap).
- **Timeouts** — inactive 45s, active 60s, handshake 60s poll / 180s age.
- **Human confirmation** — killswitch disable phrase; uninstall extra-var;
  live-test consent flag; bootstrap become on TTY.
- **Ambiguous state** — exactly-one active VPN; ambiguous profile names
  rejected; empty expected endpoints with leftover project rules fail
  coverage.

```text
Evidence:
- roles/airvpn_client/files/lib/airvpn-common.sh (lock, waits)
- roles/airvpn_client/files/airvpn-switch
- roles/airvpn_client/files/airvpn-firewall-sync
- docs/KNOWN_LIMITATIONS.md
```

---

## Engineering Decisions and Trade-offs

1. **firewalld policy objects instead of raw nftables.** Uses Fedora’s
   firewall manager and zone binding via NM. Cost: policy name length 18;
   no Ansible policy module → `firewall-cmd` + `changed_when`.
2. **Numeric endpoints only.** Closes pre-tunnel DNS. Cost: operators must
   export IP endpoints from AirVPN.
3. **Persistent allowlist of all imported endpoints.** Handshakes work
   after switch without rewriting firewall every time. Cost: UDP to every
   imported AirVPN IP:port is always excepted on underlay.
4. **Disconnect-then-activate on switch.** Never two managed tunnels;
   underlay stays REJECT in the gap. Cost: brief connectivity loss even
   for “good” switches (by design).
5. **Inactive after import.** Install cannot leave an unexpected tunnel
   up. Cost: extra disconnect logic because NM may auto-activate on
   import.
6. **Effective `ip route get`, not main-table default.** Matches NM
   policy routing. Cost: more subtle checks; documentation-range probes.
7. **Atomic: instruct or layer, never DNF, never auto-reboot.** Respects
   ostree. Cost: two-step bootstrap after layering.
8. **Uninstall restores `public`, not original zone.** Simpler; no
   pre-install snapshot of zones. Cost: metadata drift (live confirmed).
9. **Keep managed configs on uninstall by default.** Avoid accidental
   secret deletion. Cost: keys remain on disk.
10. **Live tests opt-in, CI static.** Prevents GitHub-hosted accidental
    firewall mutation. Cost: CI cannot prove kill-switch traffic.
11. **Direct-on-host Ansible only.** Finite v0.1.0. Cost: no remote
    controller.
12. **Agent allowlists that admit they are not a sandbox.** Reduces
    accidental `sudo nmcli` from coding agents; does not replace OS
    permissions.

---

## Key Technical Learnings

Project-specific, from this tree:

- **Kill-switch correctness is a mutation order problem.** Policies must
  exist before import/sync; legacy names must be deleted only after
  replacements exist; reload only after `--check-config`.
- **The VPN bootstrap problem is real.** Endpoint exceptions and numeric
  IPs exist because the underlay cannot be fully sealed if the handshake
  has nowhere to go, and DNS-before-tunnel is a leak class of its own.
- **NM profile zone ≠ firewalld runtime zone.** `connection.zone` is
  metadata; active devices need reapply + `--get-zone-of-interface`.
- **“Default route” is not one table.** Full-tunnel WireGuard on Fedora
  often needs `ip route get` / policy rules, or verification lies.
- **Ownership must be named.** Sync that deletes “any UDP rich rule”
  would vandalize admin policy. Project-shaped rules + state file is the
  actual design.
- **Fail-closed is a family of behaviors**, not a boolean: no-VPN block,
  switch gap, handshake abort, and explicit disable are different.
- **Live proof is narrower than support claims.** Silverblue + vEth
  proves that path; Wi-Fi/suspend remain implemented-not-proven.
- **Uninstall that sources installed files is not repeatable.**
  Live-found: second uninstall needed the role copy from git.
- **Immutable OS packaging is a control plane.** Atomic hosts made
  “just dnf install” the wrong abstraction.
- **Unsafe operations should require explicit operator intent.** Kill-switch
  disable requires a confirmation phrase (or explicit `--force`), while uninstall
  requires an explicit confirmation variable; neither is an implicit default path.

---

## Historical Evolution

Compressed from `main` (2026-08-02 → 2026-08-04), not a commit dump.

| Step | What changed technically |
| --- | --- |
| `b98f69a` | Initial Ansible + Bash kill-switch implementation |
| Bugbot / agent safety | Cursor deny-lists, `validate-safe`, Gitleaks full history |
| firewalld | Permanent-only zones; policy names shortened to 18 chars; legacy cleanup after new policies; `--check-config` before reload |
| WireGuard import | IFNAMSIZ-safe temp filenames; leave profiles inactive |
| Routing verify | `ip route get` / policy-routing instead of main-table default |
| Bootstrap | `--ask-become-pass`; refuse `sudo ./bootstrap.sh` |
| Integration | Opt-in VM runner; public-IP probe classification |
| Uninstall | Snapshot collector; repeatable uninstall via repo `airvpn-common.sh`; atomic audit publish + SIGINT handling |
| Docs | KNOWN_LIMITATIONS, INTEGRATION_TESTING, ROADMAP, v0.1.0 notes |
| `2d1a5d9` + tag | Merge PR #1; signed `v0.1.0` |

Several “fix:” commits are **live- or review-discovered** defects
(inactive import, policy length, routing false negatives, second
uninstall, incomplete audit publish), not drive-by style.

---

## Evidence Index

| Area | Paths |
| --- | --- |
| Role / defaults | `roles/airvpn_client/defaults/main.yml`, `tasks/*.yml` |
| Firewall | `tasks/firewall.yml`, `files/airvpn-firewall-sync`, `files/airvpn-killswitch` |
| NM | `tasks/networkmanager.yml`, `files/airvpn-import`, `files/airvpn-switch`, `files/airvpn-protect-connection` |
| Shared lib | `files/lib/airvpn-common.sh`, `airvpn-select-physical-uuids.sh` |
| Verify | `files/airvpn-check`, `files/airvpn-status`, `tasks/verify.yml` |
| Atomic | `tasks/detect.yml`, `tasks/dependencies_fedora_atomic.yml` |
| Bootstrap | `bootstrap.sh`, `requirements-controller.txt`, `requirements.yml` |
| Uninstall | `playbooks/uninstall.yml` |
| Live test | `tools/integration-test-vm`, `tests/integration/lib/*` |
| Static | `tools/validate-safe`, `tests/unit/*` |
| CI | `.github/workflows/{lint,syntax,secret-scan}.yml` |
| Limits | `docs/KNOWN_LIMITATIONS.md`, `ROADMAP.md`, `SECURITY.md` |
| Authorship | `git log`; tag `v0.1.0` |

Git history used: `git log --oneline`, tags, author stats, first/last
dates, selected fix subjects. Not a substitute for the current tree.

Web search: **not used**. Platform behavior (firewalld policy objects,
IFNAMSIZ, rpm-ostree) is taken from this repository’s code and docs.

---

## Limitations and Non-Claims

Do not use this repository as evidence for:

- Affiliation with AirVPN, AirVPN API, or account automation.
- A guaranteed leak-proof, formally verified, or “production-grade VPN
  security product.”
- Prevention of **all** leaks (all imported endpoint UDP remains allowed;
  LAN underlay is blocked by default; other interfaces/profiles may
  exist).
- Live proof on physical Wi-Fi, suspend/resume, dual uplink, tethering,
  captive portals, or every Fedora edition.
- That newly created Wi-Fi/Ethernet profiles are automatically protected.
- That CI proved real firewalld/NM/WireGuard traffic.
- That Ansible check mode proved leak resistance.
- That `airvpn-check --offline` sent packets (it inspects configuration).
- That uninstall restores original NM zones or removes all secrets.
- That `airvpn-killswitch disable` is anything other than an intentional
  underlay ACCEPT.
- That Cursor files sandbox the agent.
- That pytest is the test suite (it is unused).

Keep the live-validation sentence exact:

> Live validated on Fedora Silverblue 43 under KVM/Fedora Boxes with a
> virtual Ethernet interface.

---

## Derived Defensible Experience Statements

Bounded restatements of the evidence. Invalid if the limits above are
dropped.

- Implemented a Fedora-local Ansible + Bash AirVPN WireGuard workflow
  that uses firewalld **HOST→zone policy objects** (underlay REJECT, VPN
  ACCEPT) rather than a custom nftables tree.
- Added validation that **rejects hostname WireGuard endpoints** so
  pre-tunnel DNS is not required on a fail-closed underlay.
- Designed endpoint rich-rule sync with an **owned-rules state file** so
  unrelated administrator rules on the same policy are not deleted.
- Integrated NetworkManager import with IFNAMSIZ-safe temp files,
  `autoconnect no`, DNS priority/`~.` routing domains, and **forced
  inactive** profiles after import.
- Implemented VPN switching that disconnects all managed tunnels, waits
  for handshake, and **fails closed** (down + abort) if handshake
  verification fails, while leaving project underlay REJECT in place.
- Added `airvpn-killswitch` that mutates **only project policies** and
  requires an explicit confirmation phrase to set underlay ACCEPT,
  without stopping firewalld.
- Implemented Fedora Atomic handling that refuses DNF host-image
  mutation, detects `/run/ostree-booted`, and will not reboot after
  optional `rpm-ostree` layering.
- Separated proof levels in tooling: static `validate-safe`, advisory
  Ansible `--check`, opt-in disposable-VM lifecycle with leak probes, and
  installed-state `airvpn-check`.
- Live-validated the lifecycle on **Fedora Silverblue 43** in a KVM guest
  with virtual Ethernet, including fail-closed without VPN, switch
  probes, disconnect, and repeatable uninstall — and documented that
  physical Wi-Fi and suspend/resume are **not** in that evidence set.
- Treated WireGuard private keys as secrets: out-of-repo sources,
  root-owned copies, gitignore/gitleaks, and redacted status output —
  without claiming a full secret-management platform.

Those statements remain invalid if live-vs-static scope, AirVPN
non-affiliation, or the leak-resistant (not leak-proof) bound is omitted.
