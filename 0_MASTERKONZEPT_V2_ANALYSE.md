# 0_MASTERKONZEPT_V2_ANALYSE.md
Version: v0.3
Datum: 2026-01-07
Status: (Hybrid-Dashboard + Task-Referenzen)

## Zweck dieser Datei (WOFÜR ist sie da?)
Diese Datei ist die zentrale **Abarbeitungsgrundlage** für Masterkonzept v2.0.

HYBRID-REGEL:
- Diese Datei = **Dashboard/Backlog** (Issues, Prioritäten, Status, Decisions, Abnahmekriterien in Kurzform).
- Detailarbeit = ausgelagerte Task-Dateien unter: `tasks/T/` (Input → Output → Tests → Dependencies → Randfälle).

Ordner-Erweiterung (neu):
- `tasks/T/` enthält T01..T15 als eigenständige Spezifikationen.

## Status-Konventionen
- TODO = nicht begonnen
- IN_PROGRESS = in Arbeit
- BLOCKED = blockiert (Dependency/Decision fehlt)
- DONE = abgeschlossen (inkl. Tests/Acceptance erfüllt)
- PARKED = bewusst ignoriert (für später geparkt)

## Arbeitsregeln (Kurz)
- Keine Neuinterpretation deiner Ziele.
- Alles muss **deterministisch** und **testbar** sein.
- Aussagen sind getrennt in: **Fakten / Annahmen / Schlussfolgerungen**.
- Diese Datei ist die Arbeitsgrundlage zur Abarbeitung (Tasks + Decisions + Tests).

---

# Health Score (aktueller Stand)
BLOCKER: 1
HIGH: 1
MEDIUM: 1
LOW: 1
PARKED: 1

Risiko-Summary:
Mehrere Kernstellen bleiben unpräzise (DB-Schema/Constraints, Locking/Apply-Reconcile),
was deterministisches Enforcement und Betriebssicherheit gefährdet.

---

# Fakten / Annahmen / Schlussfolgerungen (für diese Analyse)

## Fakten (direkt aus den Dateien belegbar)
1) DB-Tabellen/Fields für Onboarding/AAA werden konzeptionell benannt, aber ohne SQL-Schema (Typen, Constraints, Indizes, FKs).
2) DNS-Enforcement via DNAT53 deckt klassisches DNS (Port 53) ab; DoH/DoT wird in dieser Runde nicht behandelt (PARKED).
3) Reconcile ist Pflicht (Timer), Locking wird als optional geführt → Concurrency-Risiko ist real.
4) Durable Accounting-Spool ist konzeptionell vorgesehen, aber ohne Quota/Retention/Overflow-Verhalten.
5) Service-IP 10.77.0.1/32 als stabiler Anker ist definiert; abarbeitbare Scope-Matrix + nft-Regel-Checkliste ist noch nicht vollständig präzisiert.

## Annahmen (noch NICHT kanonisch beschlossen)
A1) Ressourcen-Budgets (z.B. 2 vCPU / 4GB RAM / 50GB Disk) sind in der bisherigen Analyse erwähnt,
    aber (Stand jetzt) nicht als verbindliche Assumption/Decision in `03_ASSUMPTIONS_AND_RULES.txt` festgezogen.
    => Bis zur Entscheidung: nur Arbeitshypothese.

## Schlussfolgerungen (aus Fakten + Zielmodell)
S1) Ohne verbindliches SQL-Schema + Constraints ist „Panel als Zentrale“ nicht deterministisch umsetzbar (Risk: Lockouts/Inkonsistenzen).
S2) Ohne verbindliches Locking/Retry/Idempotenz kann Apply/Reconcile Drift erzeugen (Partial State, Flapping).

---

# Issue-Liste (mit Belegen) + Task-Verknüpfung

## Issue 1 — [BLOCKER] Fehlendes DB-Schema und Constraints für Onboarding-/AAA-Daten
Referenz: `blocks/B155_ONBOARDING_GATES_STATE_MACHINE.txt` (Abschnitte 5.1–5.2)

Problem:
- Pflichtfelder für Tabellen (vpn_connections / customers) werden als Liste genannt, aber ohne:
  - Datentypen
  - NOT NULL / DEFAULT
  - UNIQUE
  - CHECK (State-Enum)
  - Foreign Keys
  - Indizes
- Dadurch sind Gate#1 (Claim), Gate#2 (Verify), Subaccounts-per-Device, SimUse=1 und Accounting nicht stabil erzwingbar.

Fix (konzeptuell):
- SQL-first: vollständiges DDL + versionierte Migrationen.
- Constraints/Indizes als „Policy in SQL“.

Fehlende Tests/Acceptance (MUST):
- Migration läuft reproduzierbar (clean DB).
- UNIQUE: `vpn_connections.username` und `vpn_connections.fixed_ip`.
- CHECK: status nur erlaubte Werte.
- FK: `vpn_connections.customer_id` referenziert `customers.id`.
- Negativtests: verwaiste Claims; doppelte IP; ungültiger Status.

Task-Verknüpfung:
- T01 (Core Schema)
- T02 (RADIUS SQL Schema + Mapping)
- T07 (Gates State Matrix / Deterministische Ableitung)

---

## Issue 2 — [PARKED] DoH/DoT (aktuell nicht behandelt)
Referenzen:
- `blocks/B210_DNS_ENFORCEMENT_DNAT53.txt` (Hinweis: DNAT53 deckt nur Port 53 ab)

Status:
- Dieser Punkt wird **aktuell bewusst ignoriert / nicht behandelt** (kein Scope für v2 in der aktuellen Arbeitsrunde).

Konsequenz:
- DNS-Zwang bezieht sich in dieser Runde nur auf **klassisches DNS (Port 53)**.
- DoH/DoT bleibt als potenzieller Bypass **bekannt**, wird aber nicht als Issue/Blocker verfolgt.

Später (wenn reaktiviert):
- Task-Verknüpfung: T04, T15

---

## Issue 3 — [HIGH] Optionales Locking → Race Conditions (Apply/Reconcile Concurrency offen)
Referenzen:
- `blocks/B166_POLICY_APPLY_RECONCILE.txt` (Locking optional, Exit-Code TEMPFAIL_LOCKED)
- `blocks/B167_WIRING_HOOKS_SYSTEMD.txt` (Locking via flock optional)

Problem:
- Reconcile läuft regelmäßig; ip-up und Panel-Events können ebenfalls Apply triggern.
- Ohne verbindliches Locking/Retry/Backoff/Idempotenz sind Partial States möglich.

Fix (konzeptuell):
- Verbindliches Concurrency-Modell:
  - Single-writer Apply
  - definierte Retry/Backoff-Regeln
  - Idempotenz-Garantien
  - klare Semantik für TEMPFAIL_LOCKED

Fehlende Tests/Acceptance:
- Parallel-Trigger-Test (ip-up + Panel-Event + reconcile) → deterministischer Endzustand.
- Apply 2x → identischer State.
- Lock busy → definierter Rückgabepfad + definierte Retry Strategie.

Task-Verknüpfung:
- T03 (Locking/Retry/Idempotenz)
- T06 (nft/tc Scopes – Umsetzungsebene)
- T09 (SimUse/Stale E2E – Regression)

---

## Issue 4 — [MEDIUM] Durable Accounting Spool ohne Quota/Retention/Overflow/Replay-Safety
Referenz: `blocks/B050_SQL_AAA_RADIUS.txt` (Spool-Konzept)

Problem:
- Spool ist vorgesehen, aber ohne:
  - Max size / Quota
  - Rotation/Retention
  - Verhalten bei Disk-Full
  - Replay-Idempotenz/Dedupe

Fix (konzeptuell):
- Spool-Policy + Overflow-Verhalten + Monitoring.
- Replay-Safety (Dedup-Keying) definieren.

Fehlende Tests/Acceptance:
- DB down simulieren → spool wächst kontrolliert.
- DB up → replay ohne Duplikate.
- Disk-pressure → definierte Degradation + Alarm.

Task-Verknüpfung:
- T05 (Spool Policy)
- T15 (Monitoring/Retention/Alarmierung)

---

## Issue 5 — [LOW] Service-IP 10.77.0.1 Bindung ohne präzise Scope-Matrix + nft-Regel-Checkliste
Referenz: `000_MASTER-KONZEPT v2.0 (ZEILENFEST).txt` (Service-IP als stabiler Anker)

Problem:
- Service-IP/Netze sind definiert, aber die konkreten Scopes (Ports/Netze/Restricted-Allowlist) sind nicht vollständig als abarbeitbarer Regelkatalog beschrieben.

Fix (konzeptuell):
- Scope-Spezifikation für 10.77.0.1:
  - User vs Admin vs Restricted
  - WAN-Stealth bleibt intakt
  - Logging/Counter Tests

Fehlende Tests/Acceptance:
- WAN Scan: nur VPN Ports sichtbar.
- VPN User: nur erlaubte Services erreichbar.
- Restricted: nur Allowlist.

Task-Verknüpfung:
- T06 (nft Service-IP Scopes)
- T08/T12 (Panel/Webstack Scope)

---

# Fehlende Themen (aktuell genannt, aber noch nicht „abarbeitbar“ ausgearbeitet)
Jedes Thema braucht mindestens: Deliverables + Acceptance Tests + Dependencies.

- DB-Migrationsskripte + RADIUS-Attribute-Mapping (radcheck/radreply/radacct) inkl. Indizes. -> T02
- Logging/Monitoring/Retention-Strategie (nftables/adguard/unbound/ppp/strongSwan) + Alarmierung. -> T15
- Backup/Restore/Disaster-Recovery Runbook (DB + Spool + Configs). -> T14
- Panel-Sicherheitskonzept (Rate-Limits, CSRF, AuthZ, Scope) -> T08
- MTU/MSS/Offload: konkrete Werte + XP/Vista Tests -> T13

---

# Decision Log (muss festgezurrt werden, sonst BLOCKED-Risiko)
(Decision IDs sind Platzhalter; endgültige IDs nach deinem Decision-Record Schema)

- D001_DOH_DOT_SCOPE:
  Frage: DoH/DoT Scope (Bypass-Prevention) – wird in dieser Runde nicht behandelt
  Needed-by: T04, T15
  Status: PARKED (bewusst ignoriert)

- D002_RESOURCE_BUDGETS:
  Frage: Welche Ressourcen-Budgets gelten verbindlich (CPU/RAM/Disk)?
  Needed-by: T15 (Retention/Monitoring), T05 (Spool Quota), T12 (Webstack)
  Status: TODO

- D003_PASSWORD_STORAGE_MODEL:
  Frage: Wie werden PPP/RADIUS Credentials gespeichert (Cleartext vs Hash vs kompatibler Mechanismus)?
  Needed-by: T02, T08
  Status: TODO

- D004_RESTRICTED_VS_REJECT_POLICY:
  Frage: Welche Zustände führen zu RADIUS Reject (DISABLED) vs „Restricted via nft/tc“?
  Needed-by: T02, T07, T06
  Status: TODO

- D005_PANEL_SCOPE_USER_ACCESS:
  Frage: Darf User-Panel/Status-Seiten im User-Netz erreichbar sein (oder nur Admin-Netz)?
  Needed-by: T06, T08, T11, T12
  Status: TODO

---

# Abarbeitungs-Backlog (Index + Links)
Pfad-Regel: Task-Dateien liegen unter `tasks/T/`.

| Task | Titel | Prio | Status | Dependencies | Kurz-Output | Kurz-Tests | Link |
|------|------|------|--------|--------------|------------|-----------|------|
| T01 | DB Schema Core (customers + vpn_connections) | MUST | TODO | — | `schema/0001_core.sql` + DDL-Tests | UNIQUE/FK/CHECK + Negativtests | tasks/T/T01_DB_SCHEMA_CORE.txt |
| T02 | FreeRADIUS SQL + Mapping (radcheck/radreply/radacct) | MUST | TODO | T01, D003, D004 | `schema/0002_radius.sql` + mapping doc | Accept/Reject/Accounting/SimUse | tasks/T/T02_RADIUS_SCHEMA_MAPPING.txt |
| T03 | Policy Apply/Reconcile Locking & Concurrency | MUST | TODO | T01 | locking model doc | parallel-trigger regression | tasks/T/T03_POLICY_LOCKING_CONCURRENCY.txt |
| T04 | DoH/DoT Scope (aktuell ignoriert) | PARKED | PARKED | — | — | — | tasks/T/T04_DOH_DOT_SCOPE_DECISION.txt |
| T05 | Accounting Spool Policy (Quota/Retention/Overflow/Replay) | MUST | TODO | D002 | spool policy doc | DB-down replay + disk-pressure | tasks/T/T05_ACCOUNTING_SPOOL_POLICY.txt |
| T06 | nftables Service-IP Scopes (User/Admin/Restricted) | MUST | TODO | T03, D005 | scope spec + rule checklist | WAN scan + VPN access tests | tasks/T/T06_NFT_SERVICE_IP_SCOPES.txt |
| T07 | Gates State Matrix (Claim/Verify/Quota/Expiry) | MUST | TODO | T01, D004 | state matrix doc | deterministische Ableitung | tasks/T/T07_GATES_STATE_MATRIX.txt |
| T08 | Panel Security Model (AuthN/AuthZ/CSRF/RateLimit) | MUST | TODO | T01, D005 | security model doc | negative authz tests | tasks/T/T08_PANEL_SECURITY_MODEL.txt |
| T09 | SimUse=1 + Stale Session E2E Tests | MUST | TODO | T02, T07 | e2e testplan doc | ghost-session recovery | tasks/T/T09_SIMUSE_STALE_E2E_TESTS.txt |
| T10 | DNS Failure Modes + Healthchecks (Unbound/AdGuard/DNAT53) | MUST | TODO | T06 | failure-mode spec | stop/start + /diag signals | tasks/T/T10_DNS_FAILURE_MODES_HEALTHCHECKS.txt |
| T11 | /diag Endpoint Spec (User/Admin Sicht) | SHOULD | TODO | T06, T08 | diag spec doc | scope correctness | tasks/T/T11_DIAG_ENDPOINT_SPEC.txt |
| T12 | Webstack Hardening (OpenResty/PHP-FPM/DB) intern-only | MUST | TODO | T06, T08, D002 | hardening checklist | WAN unreachable + scope tests | tasks/T/T12_WEBSTACK_HARDENING_INTERNAL_ONLY.txt |
| T13 | MTU/MSS/Offload Defaults + XP/Vista Tests | SHOULD→MUST | TODO | Basis-VPN stabil | defaults doc | large-transfer stability | tasks/T/T13_MTU_MSS_OFFLOAD_DEFAULTS_TESTS.txt |
| T14 | Backup/Restore/DR Runbook | MUST | TODO | T01/T02/T05 | DR runbook doc | restore drill success | tasks/T/T14_BACKUP_RESTORE_DR_RUNBOOK.txt |
| T15 | Logging/Monitoring/Retention/Alerts | MUST | TODO | D002, T05, T06 | retention+alert spec | rotation + alarm simulation | tasks/T/T15_LOGGING_MONITORING_RETENTION_ALERTS.txt |

*Hinweis zu T04:
- DoH/DoT wird aktuell nicht behandelt (PARKED).

---

# Change Control (wie wir diese Datei pflegen)
- Bei Task-Start: Status in Tabelle auf IN_PROGRESS setzen.
- Bei Task-Abschluss: Status auf DONE, Tests/Acceptance kurz vermerken (1–3 Zeilen).
- Bei jeder neuen Decision: Decision Log aktualisieren.
- Changelog unten fortschreiben (nichts löschen, nichts kürzen).

---

# CHANGELOG (niemals kürzen/entfernen)
- v0.3 (2026-01-07): Task-Links auf .txt umgestellt; DoH/DoT als PARKED markiert (aktuell ignoriert); Health Score angepasst.
- v0.2 (2026-01-06): Hybrid-Dashboard eingeführt (Backlog-Tabelle + Decision Log + Task-Links), Issue-Liste in abarbeitbare Struktur überführt, Assumptions/Decisions sichtbar gemacht.
- v0.1 (Altbestand): Health Score + Issues 1–5 + Fehlende Themen + Budget-Hinweis (ohne Hybrid-Links/Decisions).