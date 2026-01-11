# xp-vpn-stack

A **Debian 13** “from scratch” **L2TP/IPsec (strongSwan + xl2tpd/pppd)** VPN stack designed specifically for **Windows XP / Vista** clients.

Goal: **Full-Tunnel**, minimal client configuration, deterministic server-side policy, and a production-ready operational model (AAA, accounting, QoS, DNS stack, internal webpanel, blockpage MITM, onboarding: Verify-Wall + Claim Token + UNCLAIMED Grace/Overdue).

---

## What this is

This repository is a **specification** (not an installer script yet).  
It is structured as:

- **MASTER (zeilenfest)**: canonical, versioned master concept
- **0_MASTERKONZEPT_ANALYSE.md**: Arbeits- und Abarbeitungsdatei (Hybrid), verweist auf Blocks/Decisions/Tasks (Diese Datei ist optional)
- **blocks/**: implementation blocks (B010–B343) defining the system precisely
- **decisions/**: decision records (ADR) explaining *why* choices were made
- **tasks/**: optional workspace files (non-canonical; the specification lives in MASTER/blocks)
- **templates/**: templates for adding future blocks/decisions consistently


The intended outcome is a reproducible server setup where the VPN software provides the tunnel, while **Linux enforces the policies** (nftables / tc / service binding).

---

## Core objectives

- **Windows XP/Vista built-in client compatibility** (no SoftEther client)
- **Full-Tunnel by design** (all client traffic exits through the VPN)
- **“Idiotensicher” client experience**
  - client should only need **FQDN + VPN-credentials (PPP) + PSK**
  - no manual IP/gateway/DNS entry (PPP/IPCP)
- **Two separated VPN networks**
  - User-Net: `10.77.10.0/24` (restricted)
  - Admin-Net: `10.77.20.0/24` (privileged)
  - Service-IP on loopback: `10.77.0.1/32` (stable anchor for internal services)
- **Strict Peer-Isolation** (user clients cannot reach each other)
- **Server exposure minimized**
  - WAN: only IPsec ports (UDP 500/4500 + ESP)
  - UDP 1701 (L2TP) only accepted **via IPsec/XFRM**
- **Per-client QoS**
  - shaping per `pppX` using `tc` (CAKE/fq_codel)
- **DNS stack (enforced)**
  - Unbound local resolver + AdGuardHome (internal)
  - **DNS enforcement is MUST** (DNAT TCP+UDP 53 from `ppp*` → `10.77.0.1:53`)
- **Internal webstack**
  - OpenResty (Nginx+Lua) + PHP-FPM + MySQL/MariaDB + phpMyAdmin (admin-only)
  - webpanel reachable only inside VPN, bound to `10.77.0.1`
- **HTTPS blockpage with internal CA (MITM)**
  - blocked domains resolve to `10.77.0.1` (AdGuard “Custom IP”)
  - OpenResty serves HTTP/HTTPS block pages
  - XP browser target: **MyPal** with NSS trust store handling
- **Time/NTP strategy**
  - XP time problems handled (pre-/post-connect strategy; optional NTP hijack to `10.77.0.1`)
## Onboarding (v2.3)
- Verify-Wall (App-Layer): Customer PENDING sieht nach Login nur Code/Resend/Support.
- claim_token (App-Layer): Claim ordnet eine VPN-Connection einem Customer zu (claim_token als Besitznachweis; nicht für VPN-Login).
- UNCLAIMED Grace/Overdue (Kernel): 30 Tage ab Provisioning/Erstellung Internet frei; danach UNCLAIMED_OVERDUE -> "Nur Panel-Zugriff" (Walled Garden), damit Verify+Claim weiterhin möglich sind.
- Hard-Stop gegen Leichen (Standardbetrieb): claim_deadline immer gesetzt (180 Tage). Nach Ablauf unclaimed -> DISABLED (kein VPN/kein Panel).


---

## Design principle: “VPN is only the tunnel – policies live in Linux”

The stack intentionally avoids relying on “VPN software features” for control:

- Routing/NAT/segmentation: **Linux**
- Policy enforcement: **nftables**
- QoS: **tc**
- Services: bound to `10.77.0.1` and firewall-limited
- AAA/accounting: **SQL + FreeRADIUS** (source of truth)
- Determinism: **policy-apply + reconcile** (drift-safe)

This is built for selling/supporting XP systems with minimal support overhead and predictable behavior.

---

## Stack overview

### VPN (XP/Vista compatible)
- strongSwan (IKEv1/IPsec, including legacy crypto requirements)
- xl2tpd + pppd
- IP assignment via PPP/IPCP (no client-side static IPs)

### Policy / Security
- nftables WAN stealth + XFRM-only L2TP
- full-tunnel forward + NAT
- MSS clamping / MTU stability measures
- outbound abuse blocking (SMTP + SMB/NetBIOS)
- IPv6 disabled (provider + OS)

### QoS
- per-PPP interface shaping via tc
- default limits by group (User/Admin) and DB-driven parameters

### DNS
- Unbound (local-only)
- AdGuardHome on `10.77.0.1:53` (UI admin-only)
- **DNS enforcement MUST**: DNAT TCP+UDP 53 from `ppp*` → `10.77.0.1:53`

### Webpanel & Blockpage
- OpenResty + PHP-FPM + DB
- panel and diagnostics endpoints internal-only
- MITM HTTPS blockpage using an internal CA
- MyPal NSS trust store integration

### Reliability / Operations (v2.0 hardened)
- systemd auto-restart + healthchecks for key services
- DPD + LCP echo to avoid stale sessions
- offload tuning defaults for virtualized environments
- **Accounting collector + session mapping** (pppX → connection_id)
- **Stale-session janitor + on-demand janitor in login flow** (SimUse lockout-safe)
- **Spool/Retention safety (v2.3)**: SQL-first settings + local safety ceilings (hard max bytes/age). On ceiling hit: **Ring Buffer (Drop Oldest) MUST** + alert (prevents disk-full failure).
- **Policy apply + reconcile (FLUSH+REBUILD)** to prevent drift
- **Hard Cut enforcement (v2.3)**: when a client becomes restricted, established flows are terminated via **conntrack flush** (bidirectional). Fail-safe: **PPP session kill (fail-closed)** if flush fails.

---

## Repository structure (authoritative)

- `00000_Ordnerstruktur.txt` : aktuelle Ordner-/Dateistruktur (Single Source of Truth für den Tree)
- `000_MASTER-KONZEPT vX.X (ZEILENFEST).txt` : canonical “zeilenfest” spec
- `00_MASTER_vX.X.txt` : readable master summary + block map
- `0_MASTERKONZEPT_ANALYSE.md` : Abarbeitungs-/Arbeitsdatei (Hybrid: führt durch Blocks/Decisions/Tasks) (optional)
- `01_CHANGELOG.txt` : version history / deltas
- `02_GLOSSAR.txt` : glossary
- `03_ASSUMPTIONS_AND_RULES.txt` : bindende Annahmen & Regeln
- `04_BLOCK_INDEX.txt` : block index (B010–B343)
- `blocks/` : implementation blocks
- `decisions/` : ADR decision records
- `tasks/` : optional workspace files (non-canonical)

---

## Phased rollout model

The spec is built around phases (VPN first, then QoS/web/MITM/gates), with the key rule:
**VPN stability first, then features**.

See `B290_PHASE_PLAN_ROLLOUT` and the MASTER file for the authoritative phase plan.

---

## Status

- Spec baseline: **MASTER v2.3**
- Blocks: B010–B343 present (incl. v2.3 hardening in B150/B166/B165–B169/B330 and Fail2ban/Outcome blocks B340–B343)
- Decision records: present (incl. D008 Verify-Wall customer scope)
- Templates: present

---

## Scope / Non-goals

- This repo does **not** ship a one-click installer yet.
- DoH/DoT bypass prevention is out of scope for the “DNS enforcement” mechanism.
- “DNS enforcement” means **port 53 enforcement** (DNAT), not DoH/DoT interception.

---

## License

This project is licensed under the [MIT License](LICENSE).