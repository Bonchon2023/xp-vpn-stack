# xp-vpn-stack

A **Debian 13** “from scratch” **L2TP/IPsec (strongSwan + xl2tpd/pppd)** VPN stack designed specifically for **Windows XP / Vista** clients.

Goal: **Full-Tunnel**, minimal client configuration, deterministic server-side policy, and a production-ready operational model (AAA, accounting, QoS, DNS stack, internal webpanel, blockpage MITM, onboarding gates).

---

## What this is

This repository is a **specification** (not an installer script yet).  
It is structured as:

- **MASTER (zeilenfest)**: canonical, versioned master concept
- **blocks/**: implementation blocks (B010–B300) defining the system precisely
- **decisions/**: decision records (ADR) explaining *why* choices were made
- **templates/**: templates for adding future blocks/decisions consistently

The intended outcome is a reproducible server setup where the VPN software provides the tunnel, while **Linux enforces the policies** (nftables / tc / service binding).

---

## Core objectives

- **Windows XP/Vista built-in client compatibility** (no SoftEther client)
- **Full-Tunnel by design** (all client traffic exits through the VPN)
- **“Idiotensicher” client experience**
  - client should only need **FQDN + username/password + PSK**
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
- **DNS stack (phased)**
  - Unbound local resolver + AdGuardHome (internal)
  - optional DNS enforcement (DNAT port 53)
- **Internal webstack**
  - OpenResty (Nginx+Lua) + PHP-FPM + MySQL/MariaDB + phpMyAdmin (admin-only)
  - webpanel reachable only inside VPN, bound to `10.77.0.1`
- **HTTPS blockpage with internal CA (MITM)**
  - blocked domains resolve to `10.77.0.1` (AdGuard “Custom IP”)
  - OpenResty serves HTTP/HTTPS block pages
  - XP browser target: **MyPal** with NSS trust store handling
- **Time/NTP strategy**
  - XP time problems handled (pre-/post-connect strategy; optional NTP hijack to `10.77.0.1`)
- **Onboarding gates**
  - Gate#1: claim / link device+VPN connection to a user account
  - Gate#2: email verification hard gate
  - enforcement via **Restricted Mode / Walled Garden** (panel-only access)

---

## Design principle: “VPN is only the tunnel – policies live in Linux”

The stack intentionally avoids relying on “VPN software features” for control:

- Routing/NAT/segmentation: **Linux**
- Policy enforcement: **nftables**
- QoS: **tc**
- Services: bound to `10.77.0.1` and firewall-limited
- AAA/accounting: **SQL + FreeRADIUS** (source of truth)

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
- optional DNS enforcement (DNAT 53)

### Webpanel & Blockpage
- OpenResty + PHP-FPM + DB
- panel and diagnostics endpoints internal-only
- MITM HTTPS blockpage using an internal CA
- MyPal NSS trust store integration

### Reliability
- systemd auto-restart + healthchecks for key services
- DPD + LCP echo to avoid stale sessions
- offload tuning defaults for virtualized environments

---

## Repository structure

- `MASTER/` or root MASTER files: canonical “zeilenfest” spec
- `blocks/` : B010–B300 implementation blocks
- `decisions/` : ADR decision records
- `templates/` : templates for consistent future changes
- `CHANGELOG` : version history / deltas

(Exact filenames may vary depending on your layout, but the intent stays the same.)

---

## Phased rollout model

The spec is built around phases (VPN first, then QoS/DNS/web/MITM/gates), with the key rule:
**VPN stability first, then features.**

See `B290_PHASE_PLAN_ROLLOUT` and the MASTER file for the authoritative phase plan.

---

## Status

- Spec baseline: **MASTER v1.8 (zeilenfest)**
- Blocks: B010–B300 present
- Decision records: present
- Templates: present

---

## Scope / Non-goals

- This repo does **not** ship a one-click installer yet.
- Some features are explicitly marked optional/phased in the spec (e.g., DNS enforcement).
- DoH/DoT bypass prevention is out of scope for the “DNS enforcement” mechanism.

---

## License

Add your chosen license here (MIT/Apache-2.0/GPL/etc.).
