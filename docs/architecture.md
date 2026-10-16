# Handoff Note — net-monitor (v4, final consolidation)

**Date:** 2026-10-11
**Status:** Planning complete + field investigation complete. Ready for Phase 0.
**Nature:** Internal network operations tool.
**Next step:** Resolve Phase 0 blockers before coding.

---

## 1. Context

Health monitor for an internal Linux network. It will run in production, be used by the network team, and evolve as the infrastructure changes.

**Reference documents:**
- `plan-health-monitor.md` — original plan.
- `network-topology.md` — infrastructure taxonomy.
- `net-monitor-investigation.md` — field questions and answers.
- `alternative-handoff.md` — architectural view based on standard tools.

**Key consolidated decisions:**
1. The stack is **Prometheus + Alertmanager + Grafana + standard exporters** (node, blackbox, pve, ipmi, postgres).
2. `net-monitor` is an **identity exporter + correlator + API/CLI** — it is not "the complete monitor".
3. `net-agent` is **very reduced** — it only covers what `node_exporter` does not.
4. Persistence: **Prometheus** (retention 30–90 days). No custom SQLite.
5. SSH sessions are only **counts** — never identity.
6. Certificate renewal via Certbot is **broken** due to a base distribution package bug. Monitor with a 30-day alert.
7. Alerts via **Telegram**, routed by **Alertmanager**.
8. Grafana behind SSO (Keycloak) **with local break-glass admin**.
9. Wazuh returns in the future; `wazuh_enabled: false` in config until then.

---

## 2. What the monitor solves

- Know, in one place, whether each network service is up and has acceptable latency.
- When something goes down, **one** alert pointing to the root cause — not a cascade.
- Detect certificates nearing expiry before they become incidents.
- Know when a workstation has an abandoned SSH session (without seeing who the user is).
- Have history to answer "since when has this been down?".

---

## 3. Architecture

### 3.1 Where it runs

- **Full stack** (Prometheus, Grafana, Alertmanager, net-monitor): on `MONITOR_HOST` or a container on the `10.0.0.0/16` network.
- **Reason:** `MONITOR_HOST` has a direct route (0% loss, <0.3 ms) to `LDAP_HOST`, `KRB_HOST`, `DNS_HOST1`, `DNS_HOST2`. Ports 389 and 636 confirmed open on `LDAP_HOST`.
- **`net-agent`:** on each workstation and, if necessary, on `ZFS_HOST` and servers that need local checks.

### 3.2 Repository structure

```
net-monitor/
├── cmd/
│   ├── net-monitor/        # identity exporter + correlator + API
│   └── net-agent/          # SSH sessions + specific local checks
├── internal/
│   ├── config/
│   ├── model/
│   ├── checker/            # ldap, kerberos, sso, tls-cert
│   ├── correlate/          # dependency graph + root_cause
│   ├── api/
│   ├── alert/              # Telegram client (if bypass is needed)
│   ├── collector/          # utmp, zfs snapshot, local sensors
│   └── agenthttp/
├── deploy/
│   ├── docker-compose.yml     # Prometheus + Alertmanager + Grafana + exporters + net-monitor
│   ├── prometheus/
│   │   ├── prometheus.yml
│   │   └── alerts.yml
│   ├── alertmanager/
│   │   └── alertmanager.yml
│   ├── grafana/
│   │   ├── provisioning/
│   │   └── dashboards/
│   ├── net-monitor.service
│   └── net-agent.service
├── Dockerfile.monitor
├── Dockerfile.agent
├── config.example.yaml
├── config.yaml            # .gitignore
├── README.md
└── go.mod
```

### 3.3 Data flow

```
                    ┌──────────────────────────────────┐
                    │  Prometheus (scrape + retention) │
                    └──────────────────────────────────┘
                       ▲      ▲       ▲        ▲
       ┌───────────────┘      │       │        └──────────────┐
       │                      │       │                       │
  node_exporter         blackbox_    pve_exporter       net-monitor
  (CPU, disk,           exporter    (Proxmox)          (identity +
   temp, SMART)         (HTTPS,                        root_cause)
                        TCP, DNS,
                        SMTP, cert)
       ▲
       │
  net-agent
  (SSH sessions +
   local checks)
                    ┌──────────────────────────────────┐
                    │  Alertmanager → Telegram         │
                    └──────────────────────────────────┘
                    ┌──────────────────────────────────┐
                    │  Grafana (single pane, SSO +     │
                    │  break-glass)                    │
                    └──────────────────────────────────┘
```

### 3.4 Stack

- Go 1.22+ (only for `net-monitor` and `net-agent`)
- Prometheus, Alertmanager, Grafana
- `node_exporter`, `blackbox_exporter`, `pve_exporter`
- `ipmi_exporter`, `smartctl_exporter` (if applicable)
- `postgres_exporter` (if `POSTGRES_HOST` has Postgres exposed)
- Docker multi-stage, non-root
- `modernc.org/sqlite` **discarded** — Prometheus is the store

---

## 4. Scope

### Inside the MVP

**Adopted (we do not build):**
- Prometheus + Alertmanager + Grafana.
- `node_exporter` on all machines (CPU, disk, temp, SMART, uptime).
- `blackbox_exporter` for HTTPS, TCP, DNS, SMTP, and **certificate expiry**.
- `pve_exporter` for Proxmox.
- `ipmi_exporter` / `smartctl_exporter` where hardware exists.

**Built (`net-monitor`):**
- LDAP bind checker (LDAPS 636).
- Kerberos AS-REQ checker.
- SSO member/non-member checker (Keycloak).
- `tls-cert` checker for local files (optional, if necessary).
- Dependency correlation → `service_root_cause`.
- REST API + CLI.
- Its own `/metrics`.

**Built (`net-agent`):**
- SSH session count (active / idle for X minutes) on workstations.
- ZFS snapshot reading on `ZFS_HOST` (if PBS has no accessible API).

**Alerts:**
- Alertmanager → Telegram.
- `CertificateExpiring` rule (30 days) — **first rule to configure**.
- Rules for real `down` and `degraded`.

### Outside the MVP
- Wazuh (feature flag `wazuh_enabled: false`).
- DHCP Discover.
- Custom web UI (Grafana is the pane).
- User identification.
- Thanos/VictoriaMetrics (local retention is sufficient).

---

## 5. Config model

```yaml
interval_default: 30s
timeout_default: 5s

thresholds:
  down_consecutive: 3
  degraded_consecutive: 1

telegram:
  bot_token_env: TELEGRAM_BOT_TOKEN
  chat_id_env: TELEGRAM_CHAT_ID

ldap:
  base_dn: "dc=example,dc=org"
  bind_dn: "cn=net-monitor,ou=services,dc=example,dc=org"
  bind_pw_env: LDAP_BIND_PW
  ca_cert: "/etc/net-monitor/internal-ca.crt"

kerberos:
  realm: "EXAMPLE.ORG"
  kdc: "kdc.example.org"
  principal: "net-monitor@EXAMPLE.ORG"
  keytab: "/etc/net-monitor/net-monitor.keytab"

sso:
  url: "https://sso.example.org"
  realm: "example"
  member_test_env: SSO_TEST_MEMBER
  non_member_test_env: SSO_TEST_NON_MEMBER

prometheus:
  url: "http://prometheus:9090"   # to query state and calculate root_cause

services:
  - name: ldap-directory
    type: ldap
    host: ldap-host
    port: 636
    location: "servers"
    depends_on: []

  - name: kerberos-kdc
    type: kerberos
    host: krb-host
    port: 88
    location: "servers"
    depends_on: []

  - name: sso-keycloak
    type: sso
    location: "servers"
    depends_on: [ldap-directory, kerberos-kdc]

  - name: nfs-storage
    type: tcp
    host: nfs-host
    port: 2049
    location: "servers"
    depends_on: [ldap-directory, kerberos-kdc]

hosts:
  - name: workstation-01
    location: "workstations"
    agent_url: http://workstation-01:9101/metrics
    idle_thresholds: [30m, 60m, 120m]

  - name: zfs-host
    location: "servers"
    agent_url: http://zfs-host:9101/metrics
    zfs_snapshots:
      - dataset: "pool/home"
        account: "BACKUP"
        max_age_days: 7

certificates:
  domains:
    - app1.example.org
    - app2.example.org
    - app3.example.org
    - app4.example.org
    - app5.example.org
  warn_days: 30
```

---

## 6. What `net-monitor` exposes on `/metrics`

### Identity
```
ldap_bind_success{server="ldap-host"} 1
kerberos_asreq_success{kdc="krb-host"} 1
sso_member_accepted{realm="example"} 1
sso_non_member_rejected{realm="example"} 1
```

### Correlation
```
service_root_cause{service="nfs-storage",root_cause="ldap-directory"} 1
service_dependency_status{parent="ldap-directory",child="nfs-storage"} 1
```

### Certificates (if the `tls-cert` checker is used)
```
service_cert_days_to_expiry{service="keycloak",path="/etc/..."} 76
```

---

## 7. Correlation logic

`net-monitor` periodically queries the Prometheus API, reads the current state of each service (`probe_success` from blackbox, `node_exporter` metrics, etc.), and applies the `depends_on` graph from the config.

For each service:

1. `up` → propagates `up`.
2. `down`:
   - If any `depends_on` is `down` → `degraded`, sets `root_cause`. **Does not alert.**
   - If no dependency is `down` → real `down`. **Alerts.**
3. Dependents inherit `degraded` recursively.

It publishes `service_root_cause{service,root_cause}` back on its own `/metrics`. Prometheus scrapes it. Alertmanager uses this to suppress cascading alerts.

---

## 8. `net-agent` ↔ Prometheus contract

`net-agent` exposes `/metrics` on `:9101` with **only** what does not exist in standard exporters:

```go
// internal/model/agent.go
type AgentMetrics struct {
    Host           string
    Timestamp      time.Time
    ActiveSessions int
    IdleSessions   map[time.Duration]int   // {30m: 3, 60m: 1, 120m: 0}
    ZFSSnapshots   []ZFSSnapshot           // {dataset, account, age_days}
}
```

### Exposed metrics
```
ssh_active_sessions{host="workstation-01"} 5
ssh_idle_sessions{host="workstation-01",threshold="30m"} 3
ssh_idle_sessions{host="workstation-01",threshold="60m"} 1
ssh_idle_sessions{host="workstation-01",threshold="120m"} 0

zfs_snapshot_age_days{host="zfs-host",dataset="pool/home",account="BACKUP"} 2
```

### Immutable rule
No user identity field. If someone proposes `username`, the answer is no. Use Wazuh for auditing, if it is ever needed.

### Collection
- Sessions: parsing `utmp` or `w -h` + TTY mtime (`/dev/pts/N`). The struct has a `user` field, but it is never read.
- ZFS: `zfs list -t snapshot -o name,creation -H` or the PBS API.

### Permissions
- `net-agent` runs as an unprivileged user, with `SupplementaryGroups=utmp` (if reading `/var/run/utmp` directly).
- Alternative: shell out to `who -u`, without privileges.

---

## 9. `net-monitor` API

| Method | Endpoint | Description | Auth |
|---|---|---|---|
| GET | `/health` | API liveness | — |
| GET | `/services` | State + `root_cause` | — |
| GET | `/services/{name}` | Detail | — |
| GET | `/graph` | Topology + propagated state | — |
| GET | `/metrics` | Prometheus | — |
| POST | `/check/{name}` | Force check | `X-Token` |

**Rule:** the API **must not** require LDAP/Keycloak to respond. If SSO goes down, the monitor must keep working — that is when it is most needed.

---

## 10. Alerts via Alertmanager

### `deploy/alertmanager/alertmanager.yml` (skeleton)
```yaml
route:
  receiver: telegram
  group_by: [alertname, service]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  routes:
    - match:
        severity: critical
      receiver: telegram
    - match:
        severity: warning
      receiver: telegram

receivers:
  - name: telegram
    telegram_configs:
      - bot_token_file: /etc/alertmanager/telegram_token
        chat_id: <CHAT_ID>
        parse_mode: Markdown
```

### `deploy/prometheus/alerts.yml` (main rules)
```yaml
groups:
  - name: certificates
    rules:
      - alert: CertificateExpiring
        expr: (probe_ssl_earliest_cert_expiry - time()) / 86400 < 30
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "Certificate for {{ $labels.instance }} expires in {{ $value | humanize }} days"
          description: "Automatic renewal may be failing. Check Certbot."

  - name: services
    rules:
      - alert: ServiceDown
        expr: probe_success == 0
        for: 3m
        labels:
          severity: critical
        annotations:
          summary: "{{ $labels.instance }} is down"

  - name: workstations
    rules:
      - alert: AbandonedSession
        expr: ssh_idle_sessions{threshold="120m"} > 0
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Abandoned session on {{ $labels.host }}"
```

---

## 11. Security

- LDAP service account: `cn=net-monitor,ou=services,dc=example,dc=org` (read-only, create it).
- Bind: LDAPS 636 with internal CA (`internal-ca.crt`).
- Kerberos principal: `net-monitor@EXAMPLE.ORG` + keytab.
- Secrets via env (`LDAP_BIND_PW`, `TELEGRAM_BOT_TOKEN`, `TELEGRAM_CHAT_ID`, `API_TOKEN`).
- `net-agent` runs as an unprivileged user.
- Grafana behind Keycloak SSO **with local break-glass admin** (golden rule).
- Prometheus/Alertmanager **not** publicly exposed; only Grafana.
- `config.yaml`, `.env`, `*.keytab`, `*.crt` in `.gitignore`.

---

## 12. Discovery: certificates

During the investigation, it was discovered that **automatic Certbot renewal is broken on `WEB_HOST`**:

- `certbot.timer` is active, runs twice a day.
- But the `python3-certbot-nginx` plugin fails with `NoInstallationError`.
- **Root cause:** package mismatch in the base distribution (Debian 13/trixie).
  - `certbot` = `4.0.0-2+deb13u1`
  - `python3-certbot` = `4.0.0-2+deb13u1`
  - `python3-certbot-nginx` = `4.0.0-2` (**without** the `+deb13u1` patch)
- The nginx binary exists at `/usr/sbin/nginx`, is executable, and is running via systemd (`/system.slice/nginx.service`). The error is misleading.
- **It is not PATH, not a container, not configuration.** It is a distribution packaging bug.

### Impact
Five certificates that may expire without renewal:

| Domain | Expires on |
|---|---|
| `app1.example.org` | 2026-12-26 |
| `app2.example.org` | 2026-12-27 |
| `app3.example.org` | verify |
| `app4.example.org` | verify (already in renewal window) |
| `app5.example.org` | verify |

### Decision
**Wait for the distribution to release `python3-certbot-nginx=4.0.0-2+deb13u1`.** `net-monitor` acts as a safety net: `CertificateExpiring` alert 30 days in advance.

### Documented Plan B (if the distribution does not fix it in time)
Migrate to `--webroot`:
1. Create `/var/www/certbot`.
2. Add `location /.well-known/acme-challenge/ { root /var/www/certbot; }` to the nginx `server` blocks.
3. Edit each `/etc/letsencrypt/renewal/*.conf`: `authenticator = nginx` → `authenticator = webroot` + `webroot_path = /var/www/certbot,`.

---

## 13. Conventions

- **Code, CLI, and comments:** Portuguese (audience is the internal team).
- **README:** Portuguese.
- **Service names:** `function-name` (e.g., `ldap-directory`, `nfs-storage`).
- **`location`:** `servers` or `workstations`.
- **Commits:** `feat:`, `fix:`, `docs:`, `test:`, `refactor:`, `chore:`.

---

## 14. Phases

### Phase 0 — Blocking items
- [ ] Test ports `KRB_HOST:88`, `DNS_HOST1:53`, `DNS_HOST2:53`, `SMTP_HOST:25` from `MONITOR_HOST`.
- [ ] Create LDAP service account `cn=net-monitor,ou=services,...` (confirm whether `ou=services` exists).
- [ ] Create Kerberos principal `net-monitor@EXAMPLE.ORG` + keytab.
- [ ] Create Telegram bot + get chat ID.
- [ ] Confirm room/VLAN labels with the team.
- [ ] Decide where the full stack will run (`MONITOR_HOST` or another VM).

### Phase 1 — Foundation
- [ ] Repo structure, `go.mod`, `config.example.yaml`, `.gitignore`.
- [ ] Config loader + validation.
- [ ] `docker-compose.yml` with Prometheus, Alertmanager, Grafana, node_exporter, blackbox_exporter.
- [ ] Configure `blackbox_exporter` for the 5 HTTPS domains.
- [ ] **`CertificateExpiring` rule in Alertmanager — first alert.**
- [ ] Alertmanager → Telegram integration.
- [ ] TCP and HTTP checkers in `net-monitor`.
- [ ] API: `/health`, `/services`, `/services/{name}`.
- [ ] CLI: `status`, `check`.

### Phase 2 — Identity + Correlation + Agent
- [ ] LDAP bind (LDAPS 636) and Kerberos AS-REQ checkers.
- [ ] SSO member/non-member checker.
- [ ] Query Prometheus API for current state.
- [ ] Dependency graph + `root_cause`.
- [ ] `/graph`.
- [ ] `net-agent` with SSH session count.
- [ ] Prometheus scraping of `net-agent`.
- [ ] Alert rules for workstations.

### Phase 3 — Extras + Documentation
- [ ] ZFS snapshot checker on `ZFS_HOST`.
- [ ] `Dockerfile.monitor` + `Dockerfile.agent`.
- [ ] `net-monitor.service` + `net-agent.service`.
- [ ] Grafana dashboard (JSON) in `deploy/grafana/dashboards/`.
- [ ] README with:
  - Overview, architecture, setup.
  - How to monitor certificates and what to do if they expire.
  - Plan B for renewal via `--webroot`.
  - Design decisions (privacy by design, correlation, break-glass).

### Post-MVP (backlog)
- [ ] Wazuh (when it returns).
- [ ] `pve_exporter` (Proxmox).
- [ ] `ipmi_exporter` / `smartctl_exporter`.
- [ ] `postgres_exporter` (if applicable).
- [ ] Long retention (Thanos/VictoriaMetrics) if needed.

---

## 15. Golden rules

1. **Alerting is the product; dashboards are decoration.** The first component to bring up is Alertmanager.
2. **`down` only after 3 consecutive failures; `degraded` after 1.**
3. **Every check in a goroutine with `context.WithTimeout`.** Never block.
4. **Jitter ±20%.** Avoids bursts.
5. **Do not alert dependents when there is a `root_cause`.** Only the root service.
6. **Sessions: counts only. Never identity.**
7. **Do not reinvent `node_exporter`/`blackbox_exporter`.**
8. **Secrets only via env.** Config, keytabs, and certs outside Git.
9. **`net-agent` never runs as root.**
10. **`net-monitor` API does not depend on LDAP/SSO to respond.**
11. **Grafana has a local emergency admin** — the monitor must be able to see the outage it is reporting.

---

## 16. Priority order (if time runs out)

1. `docker-compose.yml` + Prometheus + Alertmanager + Grafana + node/blackbox.
2. Certificate expiry alert (5 domains) → Telegram.
3. LDAP bind checker (LDAPS 636).
4. Dependency correlation (`service_root_cause`).
5. `net-agent` with SSH sessions.
6. Kerberos AS-REQ checker.
7. SSO member/non-member checker.
8. ZFS snapshot checker.
9. `pve_exporter`.
10. Wazuh (when it returns).

Cut from the bottom up.

---

## 17. Risks and mitigations

| Risk | Mitigation |
|---|---|
| Certificates expire before Debian fixes it | 30-day alert + documented `--webroot` Plan B. |
| False positives in checks | Circuit breaker (3 failures); distinguish `down`/`timeout`/`unknown`. |
| Network overload | Minimum interval 30s + jitter. |
| LDAP/Krb credentials | Read-only service account; restricted keytab; env 0600. |
| Agent as attack vector | No privileges; endpoint only on localhost or internal network. |
| SSO goes down and Grafana is lost | Local break-glass admin in Grafana. |
| Wazuh does not return | Feature flag `wazuh_enabled: false` already planned. |
| Debian does not fix the plugin | `--webroot` Plan B ready to apply. |

---

## 18. Open items

| Item | Action |
|---|---|
| Test ports `KRB_HOST:88`, `DNS_HOST1:53`, `DNS_HOST2:53`, `SMTP_HOST:25` from `MONITOR_HOST`. |
| Create LDAP service account `cn=net-monitor,ou=services,...`. |
| Create Kerberos principal `net-monitor@EXAMPLE.ORG` + keytab. |
| Create Telegram bot + get chat ID. |
| Confirm room/VLAN labels with the team. |
| Check expiration date of certificates `app3.example.org`, `app4.example.org`, `app5.example.org`. |
| Confirm whether `ZFS_HOST` has PBS with API, or only ZFS snapshot. |
| Decide VM/host where the full stack will run. |

---

## 19. Immediate next steps

1. Resolve Phase 0 items.
2. `git init`, `go mod init github.com/<org>/net-monitor`.
3. Write `config.example.yaml` with mocked services.
4. Implement Phase 1.
5. Initial commit: `chore: bootstrap project structure`.

---

*Handoff v4 concluded. Final consolidation of field decisions, certificate investigation, and architecture based on standard tools.*
