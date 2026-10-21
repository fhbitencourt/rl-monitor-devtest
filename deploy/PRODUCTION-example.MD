# Production Readiness — rl-monitor

**Last review:** 2026-10-20
**Owner:** fhbitencourt
**Status:** in development (do not deploy)

This checklist defines what must be true **before** rl-monitor is deployed
against a real network. It is a living document: mostly unchecked at first,
filled in as phases progress.

Legend:
- [ ] pending
- [x] done
- [~] partial / decision made, execution pending

---

## 1. Pre-deployment prerequisites

Nothing here is optional. Until these are closed, rl-monitor runs only on
the developer's machine, against mocks.

- [ ] Critical service ports (KDC, DNS, SMTP) reachable from `<HOST_STACK>`.
- [ ] Read-only service account created in the directory service, bind
      tested over LDAPS with `<CA_INTERNA>`.
- [ ] Kerberos principal `<PRINCIPAL>@<REALM>` created, keytab generated
      and tested with `kinit -kt`.
- [ ] Alert bot created, chat ID obtained, test message received.
- [ ] Host for the full stack decided (`<HOST_STACK>` or a VM in
      `<REDE_INTERNA>`).
- [ ] Room/VLAN labels confirmed with the operations team.
- [ ] Certificate expiry dates for all monitored domains verified and
      recorded (kept in the private ops repo, not here).
- [ ] Snapshot backend capabilities confirmed (API vs. CLI).
- [ ] Plan reviewed with the network operations team.

## 2. Secrets and credentials

- [x] `.gitignore` covers `config.yaml`, `.env`, `*.keytab`, `*.crt`,
      `*.key`, `deploy/alertmanager/telegram_token`.
- [x] No credentials hardcoded in Go code.
- [ ] Production compose uses `*_FILE` / secrets — no password visible in
      `docker inspect`.
- [ ] Dev fallbacks removed from production config (no `:-admin` in prod).
- [ ] `LDAP_BIND_PW`, `TELEGRAM_BOT_TOKEN`, `TELEGRAM_CHAT_ID`, `API_TOKEN`
      provisioned on the production host with `0600`, correct owner.
- [ ] Keytab `0600`, readable only by the service user.
- [ ] Grafana break-glass password stored in the team vault (not in repo,
      not in host `.env`).
- [ ] Secret rotation documented (who, when, how).

## 3. Network exposure and firewall

- [ ] Prometheus and Alertmanager **not** exposed beyond `127.0.0.1` /
      internal Compose network.
- [ ] `node_exporter` bound to `127.0.0.1` or internal interface.
- [ ] `blackbox_exporter` same.
- [ ] Only Grafana exposed, and only behind SSO.
- [ ] Host firewall rules validated with the operations team.
- [ ] Compose ports checked against organization firewall policy.
- [ ] No container in `network_mode: host` without documented justification.

## 4. Alerting pipeline

- [ ] `CertificateExpiring` (30-day window) validated **in production**
      (not only in dev).
- [ ] Alertmanager → Telegram path tested in production with a real alert.
- [ ] `send_resolved: true` confirmed (resolution message arrives).
- [ ] `group_by`, `group_wait`, `group_interval`, `repeat_interval` reviewed
      with the team (avoid spam and avoid silence).
- [ ] `down` / `degraded` rules configured with the intended consecutive-
      failure thresholds.
- [ ] Root-cause suppression validated: dependents do not alert when the
      root is down.
- [ ] Test alerts (silence, resolved) verified in the Alertmanager UI.
- [ ] Runbook per alert: what to do when `CertificateExpiring` fires
      (including the `--webroot` fallback).

## 5. Grafana, SSO and break-glass

- [ ] Grafana behind the organization's SSO.
- [ ] `disable_login_form = true` in production.
- [ ] Local emergency admin **exists and was tested** with SSO disabled
      (the monitor must stay usable when the systems it watches are down).
- [ ] Break-glass procedure documented and accessible to the team.
- [ ] Prometheus datasource provisioned and validated.
- [ ] Dashboards provisioned from `deploy/grafana/dashboards/`.
- [ ] Dashboard folder name consistent with project conventions.
- [ ] `grafana.db` included in the backup plan (see §7 below).

## 6. `rl-monitor` and `rl-agent` in production

- [x] API does not depend on LDAP/SSO to respond — validate with SSO down.
- [ ] Decision recorded: `rl-monitor` runs as container or systemd unit.
- [ ] `Dockerfile.monitor` (if container) or `rl-monitor.service` ready and
      tested on the production host.
- [ ] `rl-agent.service` ready; runs as an unprivileged user.
- [ ] `SupplementaryGroups=utmp` or `who -u` — path chosen and tested.
- [ ] `rl-agent` does **not** expose user identity.
- [ ] `rl-agent` reachable only on localhost or the internal network.
- [ ] Jitter (±20%) active on all checks.
- [ ] Every check uses `context.WithTimeout` — code review item.
- [ ] Prometheus scrapes `rl-monitor` and `rl-agent` in production
      (jobs uncommented in `prometheus.yml`).

## 7. Persistence, backup and retention

- [ ] Prometheus retention (30–90 days) defined and sized for available disk.
- [ ] `prometheus-data` volume on storage with sufficient space and monitored.
- [ ] `grafana.db` and `alertmanager-data` backup scheduled.
- [ ] Production `config.yaml` (secrets stripped) backed up outside Git.
- [ ] Restore tested at least once — a backup never restored is not a backup.
- [ ] `config.yaml`, keytabs and certs kept out of Git on the host
      — confirmed.

## 8. Monitoring the monitor

- [ ] Monitor is monitored: alert if Prometheus / Alertmanager / Grafana
      goes down (dead man's switch / Alertmanager `Watchdog`).
- [ ] `node_exporter` of the stack host is scraped.
- [ ] Alert for scrape failure (`up == 0`) of the stack itself.
- [ ] Periodic liveness check of the Telegram path (e.g., daily synthetic
      alert).

## 9. Privacy by design

- [ ] No user-identity field in any metric, log or API response.
- [ ] SSH sessions are counted only (`ssh_idle_sessions`), never per-user.
- [ ] `AgentMetrics` struct does not read the `user` field — code review
      item.
- [ ] `rl-agent` and `rl-monitor` logs contain no PII (sample review).
- [ ] If a full audit tool is ever added, identity auditing stays there,
      not here.

## 10. Documentation and hand-off

- [ ] `README.md` with overview, architecture, setup, environments
      (dev vs prod).
- [ ] Section "Black-box vs white-box vs identity" (recurring question).
- [ ] Section "How to monitor certificates and what to do when one expires".
- [ ] `--webroot` fallback renewal documented.
- [ ] Design decisions recorded: privacy by design, correlation,
      break-glass.
- [ ] `config.example.yaml` up to date and matching the real contract.
- [ ] Alert runbooks linked from the README.
- [ ] Someone other than the author can bring the stack up following the
      README (hand-off test).

## 11. Deploy and rollback

- [ ] `docker-compose.prod.yml` complete and reviewed.
- [ ] Deploy documented step-by-step (exact commands).
- [ ] Rollback documented (previous version, restore volumes).
- [ ] Maintenance window agreed with the team (if applicable).
- [ ] Container `healthcheck:` configured.
- [ ] `restart: unless-stopped` on all production services.
- [ ] Log rotation configured (avoid disk fill).

## 12. First 72 hours post-deploy

- [ ] Stack stable for 72h with no unexpected restart.
- [ ] At least one real alert received and handled via Telegram.
- [ ] No recurring false positives.
- [ ] Prometheus disk usage within expectation.
- [ ] Team informed that the monitor is live and how to silence alerts.
- [ ] Post-deploy review scheduled (what worked, what to adjust).

---

## "Ready for production" criterion

rl-monitor is ready for production when:

1. Sections 1–3 are 100% complete (prerequisites and security).
2. Section 4 (alerting) is complete — alerting is the product.
3. Section 5 (break-glass) is complete — the monitor must survive the
   outage it reports.
4. Sections 7 and 11 are complete — backup and rollback exist and were
   tested.
5. Final review with the network operations team recorded.

Sections 8–10 and 12 may be completed during the first weeks of operation,
but must not remain open indefinitely.

---

## How to use this document

- Review at the end of each phase.
- Mark `[~]` when the decision is made but execution is pending.
- Add items as new risks appear; remove ones that do not apply, with
  justification in the commit message.
- Do not deploy with items from sections 1–3 or 5 open.
