Health Score
BLOCKER: 1
HIGH: 2
MEDIUM: 1
LOW: 1
Risiko-Summary: Mehrere Kernstellen bleiben unpräzise (DB-Schema, DNS-Bypass, Locking), was deterministische Enforcement und Betriebssicherheit gefährdet.

1. [BLOCKER] Fehlendes DB-Schema und Constraints für Onboarding-/AAA-Daten
- Referenz: blocks/B155_ONBOARDING_GATES_STATE_MACHINE.txt / Abschnitte 5.1–5.2 / Zeilen 45–64
- Was ist das Problem? Pflichtfelder der Tabellen vpn_connections/customers werden nur als Feldnamen skizziert, ohne Typen, Indizes, NOT NULL/UNIQUE-Constraints oder Fremdschlüsseldefinitionen.
- Warum ist das ein Problem? Ohne verbindliches Schema lassen sich weder RADIUS noch Panel eindeutig implementieren; Claim/Verify-Logik kann nicht deterministisch auf Konsistenz prüfen (z.B. Unique auf username, customer_id FK, Deadlines als DATETIME, Default-Werte), wodurch Integritätsfehler oder Lockouts entstehen.
- Beweis/Beleg aus Text: Die Liste nennt lediglich „id, username, password..., trial_until, verify_deadline...“ ohne weitere Spezifikation.
- Fix-Vorschlag (konzeptuell): Vollständiges DB-Schema (Typ, NOT NULL/NULL, UNIQUE, FK, Default, Index) für vpn_connections, customers, radcheck/radreply/radacct inkl. Claim-/Verify-Felder und Simultaneous-Use-Constraint definieren.
- Tests/Acceptance Criteria, die fehlen oder nötig sind: DB-Migrationsskript validiert; UNIQUE/NOT NULL verletzt -> deterministischer Fehler; FK-Checks verhindern verwaiste Claims; Schema-Linter/CI-Test gegen Referenz-Migration.

2. [HIGH] DNS-Zwang umgehbar via DoH/DoT ohne Gegenmaßnahmen
- Referenz: blocks/B210_DNS_ENFORCEMENT_DNAT53.txt / Abschnitt 8 / Zeilen 35–37
- Was ist das Problem? Der Block benennt DoH als Fehlerbild, beschreibt aber keinerlei technische Gegenmaßnahme (Blockierung von DoH-Zielendpunkten, TLS-SNI/JA3-Filter, HTTP-Proxy-Interception) oder Monitoring.
- Warum ist das ein Problem? Nutzer können DNS-Zwang über HTTPS umgehen und ungescannten Traffic ins Internet tunneln, was Content-Filter, Abuse-Tracking und Policy-Compliance aushebelt.
- Beweis/Beleg aus Text: „Some apps use DoH -> not covered by DNS enforcement (later policy decision).“
- Fix-Vorschlag (konzeptuell): Mindestpolicy definieren (z.B. blockiere bekannte DoH-IPs/FQDNs, erzwinge HTTP/HTTPS Proxy oder TLS-Intercept auf Port 443) inklusive Scope, Exceptions und Wartung.
- Tests/Acceptance Criteria, die fehlen oder nötig sind: Testfälle für gängige DoH-Provider (mozilla.cloudflare-dns.com, dns.google) via TCP/UDP/HTTPS; Verifikation, dass DNS-Leaks im Restricted- und Normal-Modus geblockt oder protokolliert werden.

3. [HIGH] Optionales Locking lässt Policy-Apply/Race-Condition offen
- Referenz: blocks/B166_POLICY_APPLY_RECONCILE.txt / Abschnitte 9–11 / Zeilen 147–188 sowie blocks/B167_WIRING_HOOKS_SYSTEMD.txt / Abschnitt 6 / Zeilen 94–107
- Was ist das Problem? Reconcile und Panel/ip-up-Apply teilen dasselbe Tool, aber Locking ist nur als „optional“ erwähnt; es fehlen feste Regeln zu Mutex/Retry/Backoff. Parallel ausgeführte Applies/Reconciles können widersprüchliche nft/tc-States erzeugen oder Exit-Code 4/5 liefern, ohne Recovery-Plan.
- Warum ist das ein Problem? Bei schnellen Panel-Events oder gleichzeitigen Connects kann ein Reconcile einen laufenden Strict-Apply überholen; deterministische Enforcement-Garantie (Drift-Schutz) wird verletzt und Clients bleiben im falschen Restricted-/Speed-Status.
- Beweis/Beleg aus Text: Reconcile-Timer ist Pflicht, aber Locking nur „optional“ (kein MUSS), Exit-Code 5 „TEMPFAIL_LOCKED (optional)“ ohne Ablaufdefinition.
- Fix-Vorschlag (konzeptuell): Verbindliches Locking-Konzept (flock o.ä.) als MUSS mit Timeout, Queueing oder Retry-Backoff definieren; Klarstellung, wie Panel-Events warten oder erneut ausführen, wenn Reconcile aktiv ist.
- Tests/Acceptance Criteria, die fehlen oder nötig sind: Parallel-Apply-Test (ip-up + reconcile + Panel-Event) validiert deterministische Endzustände; Lock-Timeout-Test; Wiederhol-Strategie dokumentiert.

4. [MEDIUM] Durable Accounting Spool ohne Größen-/Retention-Grenzen
- Referenz: blocks/B050_SQL_AAA_RADIUS.txt / Abschnitte 7–8 / Zeilen 77–93
- Was ist das Problem? Das Konzept verlangt persistentes Spooling bei DB-Down in /var/lib/vpn-accounting/, nennt aber keine Maximalgröße, Rotationsstrategie oder Verhalten bei vollem Datenträger.
- Warum ist das ein Problem? Bei längeren DB-Ausfällen oder Missbrauch können Spool-Dateien das 50GB-Limit des Servers füllen, was weitere Dienste (DB, Logs) lahmlegt und zu Datenverlust führt.
- Beweis/Beleg aus Text: Beschreibt nur „deltas werden ... gespult (persistentes Journal) und später nachgetragen“, ohne Limits/Rotation.
- Fix-Vorschlag (konzeptuell): Retention/Quota für Spool definieren (z.B. max Größe/Datei-Anzahl, älteste Drop/Backpressure), Monitoring-Alarm bei >80%, Graceful-Degradation-Strategie dokumentieren.
- Tests/Acceptance Criteria, die fehlen oder nötig sind: Simulation langer DB-Down-Phase mit hohem Traffic -> Spool bleibt unter definierter Größe; Alarm/Stop bei Limit; Recovery-Test bestätigt vollständige Nachtragung ohne Datenverlust oder Disk-Full.

5. [LOW] Service-IP-Bindung ohne Firewall-Ausnahme-Details für internen Webstack
- Referenz: 000_MASTER-KONZEPT v2.0 (ZEILENFEST).txt / Abschnitt 4 / Zeilen 57–65
- Was ist das Problem? Es wird gefordert, alle internen Services auf 10.77.0.1/32 zu binden, jedoch fehlen konkrete nftables-Ausnahmen/Loopback-Routen für Panels/Status/MITM-Blockpage, insbesondere wie Admin/User-Netze zugreifen dürfen und wie WAN-Stealth dies nicht bricht.
- Warum ist das ein Problem? Ohne präzise Filter-/NAT-Regeln riskieren Implementierende entweder unerreichbare Panels (zu stark gefiltert) oder unnötige Service-Exponierung (zu offen), was Stealth-Vorgaben (nur UDP500/4500/ESP sichtbar) konterkariert.
- Beweis/Beleg aus Text: Service-IP wird genannt, aber keine zugehörigen nftables Chains/Accept-Regeln für interne Zugriffe beschrieben.
- Fix-Vorschlag (konzeptuell): Firewall-Scope für 10.77.0.1 definieren (welche Interfaces/Ports dürfen zugreifen, Reihenfolge zu NAT/Stealth-Chains, Logging), inkl. Tests für User/Admin/Restricted.
- Tests/Acceptance Criteria, die fehlen oder nötig sind: Zugriffstests auf Panel/Status/Blockpage aus User/Admin-Netz funktionieren; WAN-Probing zeigt nur erlaubte VPN-Ports; Regression-Test für Restricted-Mode-Zugriffe auf 10.77.0.1.

Fehlende Themen
- Vollständige DB-Migrationsskripte und RADIUS-Attribute-Mapping (radcheck/radreply/radacct) inkl. Indizes.
- Logging/Monitoring/Retention-Strategie (nftables/adguard/unbound/ppp/strongSwan) unter 4GB RAM.
- Backup/Restore- und Disaster-Recovery-Runbook (DB + Spool + configs).
- Panel-Sicherheitskonzept (Rate-Limits, CSRF, AuthZ für Admin vs User, IP-Whitelists) jenseits der Grundnennung.
- MTU/MSS/Offload konkrete Werte und Tests auf XP/Vista bei NAT-T/Fragmentierung.

Top 10 Prioritäten
1) DB-Schema mit Constraints und Migrationen finalisieren und versionieren.
2) Verbindliches Locking/Retry-Konzept für vpn-policy-apply + reconcile definieren und testen.
3) DoH/DoT-Bypass-Strategie (Policy + Blocking) festlegen und testen.
4) Accounting-Spool-Quota/Retention + Monitoring definieren.
5) nftables-Regelwerk für Service-IP 10.77.0.1 präzisieren (Panel/Status/Blockpage).
6) Gate#1/Gate#2 Akzeptanztests (Trial/Claim/Verify) mit deterministischen States spezifizieren.
7) Stale-Session/Simultaneous-Use End-to-End Tests (B168/B169/B170) mit Fehlerszenarien dokumentieren.
8) DNS-Stack (Unbound+AdGuard) Ausfallszenarien + Fallback/Healthchecks definieren.
9) Webstack (OpenResty/PHP-FPM/DB) Hardening + Interne-Only-Erreichbarkeit inkl. nftables-Scopes festlegen.
10) Log/Metric-Retention und Ressourcen-Budgets (2 vCPU/4GB/50GB) mit Alarmierung festlegen.
