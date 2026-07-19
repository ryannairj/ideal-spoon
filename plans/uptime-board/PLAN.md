# Uptime Board — self-hosted uptime monitoring + status page in one binary

**Category:** Dev tool / self-hosted · **Difficulty:** Easy–Medium · **Platform:** Server, single binary

## 1. Vision

`uptimeboard serve` → add monitors (HTTP, TCP, ping, DNS, keyword, heartbeat) in a clean UI, get alerts where you live (ntfy, Telegram, Slack, email, webhook), and flip on a public status page with incident history — all from one Go binary + one SQLite file. Uptime Kuma proved the demand; we aim for the same niche with sharper status pages, real SLO math, and a first-class API/config-as-code story.

## 2. Why it can win

- Kuma is beloved but Node-heavy, API-poor, and its status pages are basic. Wedge: **single static binary**, full REST API + `monitors.yaml` GitOps mode, SLO/error-budget views, and status pages you'd actually show customers.
- Heartbeat (dead-man's-switch) monitors for cron jobs are underserved outside paid SaaS (Healthchecks.io) — include them natively.

## 3. Users & use cases

Homelabbers, indie SaaS founders, small ops teams.

1. Monitor 30 endpoints at 30 s intervals; get a Telegram ping on down + recovery, with failure reason.
2. Public `status.myapp.com`: component groups, 90-day bars, incident timeline with updates.
3. Backup cron sends `curl $HEARTBEAT_URL` nightly; silence > 26 h → alert.
4. `monitors.yaml` in a repo; CI applies it via API (GitOps).
5. Monthly SLO report: 99.92% vs 99.9% target, error budget burned.

## 4. MVP scope & non-goals

**MVP:** monitor types (HTTP(S) incl. keyword/JSON-path/status assertions + cert expiry, TCP, ICMP ping, DNS record, heartbeat); scheduler; alert channels (ntfy, Telegram, Slack webhook, generic webhook, SMTP) with routing + escalation delay; status pages (multiple, custom domain, component groups, manual incidents with updates); SLO targets + reports; REST API + API keys; YAML apply mode; auth (local users + TOTP).
**Non-goals:** distributed multi-region probes (v2 — schema keeps `probe_id`), APM/tracing/logs, on-call rotations/paging policies (that's an incident tool), agent-based server metrics (maybe a tiny push-gauge later).

## 5. Tech stack

- **Server:** Go 1.23, chi + embedded React UI, SQLite (WAL) via sqlc; goroutine-pool scheduler with jitter; `net/http` + `crypto/tls` for checks, `pro-bing` for ICMP (with capability note), `miekg/dns`.
- **Packaging:** static binary, Docker, systemd unit generator; SQLite backup endpoint (`VACUUM INTO`).

## 6. Architecture

```
scheduler (per-monitor ticker + jitter, concurrency cap)
  → checker (type-specific) → result{ok, latency, detail}
  → state machine per monitor: up →(N consecutive fails)→ down →(M passes)→ up
      transitions → alert router (channels, escalation, dedup, recovery notices)
  → results table (raw, ring-pruned) + rollups (minute→hour→day) for cheap 90-day charts
status pages read rollups + incidents; public pages served from same binary, cache-friendly
```

## 7. Data model

- `monitors(id, name, type, target, config_json /*assertions, timeout, interval, retries, headers, body, dns opts, expected*/, group_id, enabled, created_at)`
- `results(monitor_id, at, ok, latency_ms, detail, probe_id)` (pruned per retention) · `rollups(monitor_id, bucket, period{1m,1h,1d}, up_count, down_count, avg_latency, p95_latency)`
- `state(monitor_id, current{up,down,paused,pending}, since, last_error)`
- `heartbeats(monitor_id, slug, grace_sec, last_ping_at)` — ping URL `/hb/<slug>`.
- `channels(id, type, config_json, enabled)` · `routes(monitor_or_group_id, channel_id, on{down,recovery,cert_expiry,slo_burn}, delay_sec)`
- `status_pages(id, slug, title, custom_domain, theme_json, published)` · `page_components(page_id, monitor_or_group_id, display_name, sort)`
- `incidents(id, page_id, title, severity, status{investigating,identified,monitoring,resolved}, started_at, resolved_at)` · `incident_updates(incident_id, body_md, at)`
- `slos(monitor_id, target_pct, window{30d,90d,month})` · `users`, `api_keys`, `settings`.

## 8. Feature specs

**F1 — Checks.** HTTP: method/headers/body, follow-redirect cap, assertion chain (status in set, body keyword / JSON-path equals, max latency), TLS cert days-remaining monitor (alert thresholds 21/7/1). Retries before down (default 1 retry after 10 s). *AC:* each type covered by integration tests against local fixtures (test HTTP server with fault injection, dnsmasq container, TCP echo); a flapping endpoint honors N/M hysteresis exactly.

**F2 — Scheduler.** 500 monitors @ 30 s on a 1-vCPU box: jittered start, global concurrency cap, per-monitor timeout isolation. *AC:* load test proves < 5% tick drift at 500×30 s; one hanging target never delays others (goroutine leak test).

**F3 — Alerting.** Channel plugins with test-fire button; routing per monitor/group; escalation (notify B if still down after X min); dedup (one alert per transition); recovery includes downtime duration + reason; quiet hours per channel. *AC:* transition storms (flap) produce ≤ 1 alert per hysteresis-confirmed transition; every channel has a fixture-server test.

**F4 — Heartbeats.** Slug URL accepts GET/POST; grace window; optional "start" pings for duration tracking. *AC:* fake-clock tests for grace/late logic; ping endpoint unauthenticated but rate-limited and 404-on-unknown-slug.

**F5 — Status pages.** Component groups with 90-day bars (from rollups), overall banner (operational/degraded/major), incident timeline + subscribe-via-RSS; manual incidents with updates; theme (logo, colors, dark mode); custom domain guide. *AC:* page is static-cacheable (ETag, renders < 100 ms server-side), looks correct with 0 and 50 components, and shows *only* monitors assigned to that page (isolation test).

**F6 — SLO & reports.** Per-monitor target; dashboard shows attainment + error-budget remaining; monthly report page + optional email; `slo_burn` alert when budget burn rate > threshold. *AC:* SLO math property-tested against rollups (maintenance windows excluded when flagged).

**F7 — API + GitOps.** Full CRUD REST (OpenAPI published), API keys with scopes; `uptimeboard apply monitors.yaml` (and `export`) — declarative reconcile (create/update/delete-with-confirm). *AC:* apply→export round-trip is stable; reconcile is idempotent (second apply = no-ops).

## 9. Milestones

- **M0 (1):** binary + UI shell + HTTP monitor + scheduler + state machine + results chart.
- **M1 (2):** remaining check types, hysteresis/retries, rollups + retention. *Ships: solid monitor.*
- **M2 (3):** alert channels + routing + escalation + test-fire. *Ships: daily-driver.*
- **M3 (4):** status pages + incidents + RSS. *Ships: the showpiece.*
- **M4 (5):** heartbeats, SLOs/reports, API/GitOps, auth+TOTP, backup endpoint, docs. *Ships: v1.0.*

## 10. Testing

- Fault-injection fixture server (timeouts, resets, slow-loris, bad TLS, redirect loops) drives checker tests.
- Fake-clock everywhere (hysteresis, escalation, grace, SLO windows).
- Long-run soak in CI-nightly: 200 monitors, 24 h compressed via clock scaling, assert no leaks/drift.
- Playwright: onboarding → monitor → simulated outage → alert (to fixture webhook) → status page shows incident.

## 11. Risks & open questions

- ICMP needs CAP_NET_RAW → auto-fallback to UDP ping/TCP:80 probe with a clear UI note; document Docker flags.
- SQLite write volume at scale → batch inserts, WAL, rollup pruning; tested at 500 monitors (the stated ceiling; be honest in docs).
- Status-page custom domains + TLS → document reverse-proxy path; optional built-in autocert for the page vhost.
