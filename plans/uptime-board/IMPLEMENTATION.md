# Implementation Spec — Uptime Board

Read `PLAN.md` first. This file fixes the implementation decisions; do not re-decide them.

## 0. Fixed decisions

| Decision | Value |
|---|---|
| Name | **Uptime Board** (`uptimeboard`) · License MIT |
| Language | Go 1.23 single binary; embedded React UI; SQLite WAL (`modernc.org/sqlite`); port 3001 |
| Check libs | HTTP: stdlib + own assertion chain; ICMP: `prometheus-community/pro-bing` (fallback: TCP:80 connect probe, auto-selected on CAP error); DNS: `miekg/dns` |
| Hysteresis defaults | down after N=2 consecutive fails (1 retry after 10 s inside a tick), up after M=2 passes; per-monitor override |
| Scheduler | per-monitor `time.Ticker`-equivalent via central timing wheel; jitter = hash(id)%interval on start; global concurrency cap 50 (semaphore); per-check timeout default 15 s |
| Rollups | 1m (24 h retention), 1h (90 d), 1d (forever); raw results 48 h; rollup job every minute |
| Channels | ntfy, telegram, slack webhook, generic webhook (JSON, HMAC header), SMTP — plugin interface `Channel{Send(Event) error; Test() error}` |
| Alert semantics | one alert per confirmed transition; escalation = second route with `delay_sec` cancelled on recovery; recovery message includes downtime duration + last error |
| Heartbeat URL | `POST|GET /hb/{slug}` (slug nanoid 12); rate-limit 10/min/slug; unknown slug → 404 |
| Status pages | server-rendered (Go templates) at `/s/{slug}` + custom-domain via Host-header match; ETag caching; RSS at `/s/{slug}/rss` |
| SLO | targets per monitor; windows 30d/90d/calendar-month; maintenance windows excluded (`maintenance` table) |
| Auth | local users (argon2id) + TOTP (`pquerna/otp`); API keys with scopes `read|write` |
| GitOps | `uptimeboard apply monitors.yaml` / `export` — reconcile by monitor `name` as natural key; deletes require `--prune` |
| Backup | `GET /api/backup` (admin) → `VACUUM INTO` temp + stream |

## 1. Repository layout

```
uptimeboard/
  cmd/uptimeboard/main.go            # serve, apply, export, version
  internal/
    sched/{wheel.go, runner.go}
    checks/{check.go (iface), http.go, tcp.go, ping.go, dns.go, keyword.go (part of http asserts),
            heartbeat.go, tls.go (cert expiry)}
    state/{machine.go, hysteresis.go}
    alert/{router.go, escalate.go, dedupe.go, channels/{ntfy.go, telegram.go, slack.go, webhook.go, smtp.go}}
    rollup/{rollup.go, retention.go}
    slo/{slo.go, burn.go}
    status/{render.go, templates/, themes.go, incidents.go, rss.go, domains.go}
    api/{routes.go, auth.go, apikeys.go, gitops.go, backup.go}
    store/{schema.sql, queries.go}
  web/src/pages/{Dashboard.tsx, MonitorForm.tsx, MonitorDetail.tsx, Channels.tsx,
                 StatusPages.tsx, StatusPageEditor.tsx, Incidents.tsx, SLO.tsx, Settings.tsx}
  test/faultserver/                   # fixture HTTP server: timeout, reset, slowloris, bad TLS, redirect loop
  fixtures/{monitors.yaml, dst/}
  action-free; Dockerfile; systemd/uptimeboard.service
```

## 2. Dependencies

Go: `chi, modernc.org/sqlite, pro-bing, miekg/dns, pquerna/otp, alexedwards/argon2id, rs/zerolog, gopkg.in/yaml.v3, wneessen/go-mail (smtp)`. Web: react, tanstack-query, uplot, tailwind. Dev: vitest, playwright, `httptest` + faultserver.

## 3. Configuration

Env: `UB_DATA_DIR (default /var/lib/uptimeboard), UB_LISTEN=:3001, UB_BEHIND_PROXY, UB_PUBLIC_URL`. Everything else in DB settings. First run → setup page (admin + TOTP optional).

## 4. Database schema (key DDL)

```sql
CREATE TABLE monitors (id TEXT PK, name TEXT NOT NULL, type TEXT CHECK(type IN
  ('http','tcp','ping','dns','heartbeat')), target TEXT, config TEXT NOT NULL DEFAULT '{}',
  interval_sec INTEGER DEFAULT 60, timeout_sec INTEGER DEFAULT 15,
  fails_to_down INTEGER DEFAULT 2, passes_to_up INTEGER DEFAULT 2,
  group_name TEXT, enabled INTEGER DEFAULT 1, created_at INTEGER);
CREATE TABLE results (monitor_id TEXT, at INTEGER, ok INTEGER, latency_ms INTEGER,
  detail TEXT, PRIMARY KEY(monitor_id, at));
CREATE TABLE rollups (monitor_id TEXT, period TEXT CHECK(period IN ('1m','1h','1d')),
  bucket INTEGER, up_count INTEGER, down_count INTEGER, avg_latency REAL, p95_latency REAL,
  PRIMARY KEY(monitor_id, period, bucket));
CREATE TABLE state (monitor_id TEXT PK, current TEXT CHECK(current IN ('up','down','paused','pending')),
  since INTEGER, last_error TEXT, consec_fails INTEGER DEFAULT 0, consec_passes INTEGER DEFAULT 0);
CREATE TABLE heartbeats (monitor_id TEXT PK, slug TEXT UNIQUE, grace_sec INTEGER,
  last_ping_at INTEGER, last_start_at INTEGER);
CREATE TABLE channels (id TEXT PK, type TEXT, name TEXT, config TEXT, enabled INTEGER DEFAULT 1);
CREATE TABLE routes (id TEXT PK, target_kind TEXT CHECK(target_kind IN ('monitor','group','all')),
  target TEXT, channel_id TEXT, on_event TEXT CHECK(on_event IN ('down','recovery','cert_expiry','slo_burn')),
  delay_sec INTEGER DEFAULT 0, quiet_from TEXT, quiet_to TEXT);
CREATE TABLE status_pages (id TEXT PK, slug TEXT UNIQUE, title TEXT, custom_domain TEXT,
  theme TEXT DEFAULT '{}', published INTEGER DEFAULT 0);
CREATE TABLE page_components (page_id TEXT, ref_kind TEXT, ref TEXT, display_name TEXT, sort INTEGER);
CREATE TABLE incidents (id TEXT PK, page_id TEXT, title TEXT, severity TEXT,
  status TEXT CHECK(status IN ('investigating','identified','monitoring','resolved')),
  started_at INTEGER, resolved_at INTEGER);
CREATE TABLE incident_updates (incident_id TEXT, at INTEGER, body_md TEXT);
CREATE TABLE slos (monitor_id TEXT PK, target_pct REAL, window TEXT);
CREATE TABLE maintenance (id TEXT PK, monitor_id TEXT, from_at INTEGER, to_at INTEGER, note TEXT);
CREATE TABLE users/api_keys/settings …
```

## 5. HTTP monitor config (zod-equivalent JSON schema, stored in `monitors.config`)

`{method, headers{}, body, followRedirects: 5, assertions: [{kind: 'status_in'|'body_contains'|'json_path_eq'|'max_latency_ms', …}], tls_expiry_days: [21,7,1]}`. DNS config: `{record_type, expected[], resolver}`. Heartbeat: `{grace_sec, track_duration: bool}`.

## 6. API contract (`/api/v1`, session or API key)

Full CRUD: `/monitors` (+ `POST /monitors/{id}/pause|resume|test-now`), `/channels` (+ `/test`), `/routes`, `/status-pages` (+ components, incidents, updates), `/slos`, `/maintenance`, `/api-keys`. Reads: `/monitors/{id}/results?period=1m&from&to` (rollups), `/state`, `/slo-report?month=`. GitOps: `POST /apply` (dry-run flag → diff), `GET /export.yaml`. `GET /openapi.json` (hand-maintained YAML). Public: `/s/{slug}`, `/s/{slug}/rss`, `/hb/{slug}`, `/healthz`.

## 7. Milestone task lists

**M0** — T1 scaffold + schema + setup flow + auth; T2 HTTP check + assertion chain + timing wheel + state machine; T3 dashboard (monitor cards, latency chart from raw) + MonitorForm; T4 faultserver + check tests; T5 Dockerfile/systemd + CI.
**M1** — T1 tcp/ping(+fallback)/dns/tls-expiry checks + configs + forms; T2 hysteresis + retry-in-tick (flap test exact-transitions); T3 rollups + retention + chart switch to rollups; T4 load test 500×30 s on 1 vCPU CI job (tick drift < 5%, no goroutine leak via `goleak`).
**M2** — T1 channel interface + 5 channels + Test buttons (fixture servers each); T2 router + dedupe + quiet hours + escalation (fake clock) + recovery content; T3 routes UI. *Daily-driver.*
**M3** — T1 status page render + themes (logo/colors/dark) + ETag + component groups + 90-day bars (from 1d rollups); T2 incidents CRUD + timeline + RSS; T3 subscribe-free (RSS only, MVP decision); T4 custom-domain host matching + reverse-proxy doc; T5 page isolation test + 0/50-component render tests + <100 ms server-render bench.
**M4** — T1 heartbeats (slug endpoint, grace logic, start-pings duration, fake-clock suite, rate limit); T2 SLO math (property tests vs rollups; maintenance exclusion) + error-budget dashboard + monthly report page + slo_burn route event; T3 API keys + full OpenAPI + GitOps apply/export (round-trip stable, idempotent second apply, `--prune` confirm) ; T4 backup endpoint + TOTP + docs. Tag v1.0.

## 8. Test mapping

Fault-injection suite per check type; fake-clock everywhere (hysteresis/escalation/grace/SLO); nightly compressed soak (clock-scaled 24 h, 200 monitors); Playwright: onboarding → monitor → simulated outage (faultserver flip) → webhook alert asserted → status page shows incident.

## 9. Edge-case semantics & Error Recovery

**Assertion chain semantics (AND, no short-circuit for reporting).** All assertions in an HTTP monitor's `assertions[]` are combined with **AND** — the check passes only if every assertion passes. There is no OR in MVP (compose two monitors if you need it). Evaluation does **not** short-circuit: every assertion is evaluated so `results.detail` records *all* failures (e.g. `[status_in: ok, body_contains: FAIL, max_latency_ms: FAIL]`), which makes alerts actionable. AC: a 3-assertion monitor with 2 failing reports both failing clauses in `detail`.

**Rollup bucket-boundary assignment.** A raw result at epoch `at` belongs to bucket `floor(at / periodSec) * periodSec` (left-closed, right-open: `[bucket, bucket+periodSec)`). A result whose timestamp equals a boundary belongs to the **newer** (later) bucket. Rollup job is idempotent: it recomputes a bucket from raw results by this rule, so re-running never double-counts. AC: a result at exactly `12:01:00.000` lands in the `12:01` minute bucket, not `12:00`.

**Heartbeat start-ping vs grace window.** A monitor with `track_duration` accepts two ping kinds: `start` (sets `last_start_at`) and completion (`POST|GET /hb/{slug}`, sets `last_ping_at`). Down logic: monitor is **down** when `now - last_ping_at > interval + grace_sec`. A `start` ping does **not** reset the down timer (only a completion ping does) but records run duration = `last_ping_at - last_start_at` when the next completion arrives. An orphan `start` with no completion within `interval + grace` → down with `detail="job started but never finished"`. AC covered by fake-clock suite.

**Custom-domain TLS (no built-in autocert — preserves static-binary story).** The single binary does **not** embed ACME/autocert. Custom-domain status pages are matched by Host header and served over plain HTTP; TLS termination for custom domains is the operator's reverse proxy (Caddy/Traefik/nginx), documented in M3 T4's reverse-proxy guide. This keeps the "one static binary, no surprise outbound ACME calls" guarantee intact.

**Error Recovery & Graceful Degradation:**

| Failure | Trigger | Backoff / handling | Fallback | User-facing UX |
|---|---|---|---|---|
| Alert channel send fails | `Channel.Send` returns error or times out | retry 3×, base 5 s, ×2, cap 60 s + jitter; per-(route,event) delivery is tracked | after final retry → mark route delivery `failed`, write to `alert_log`; **does not** block other routes for the same event | dashboard shows a red "delivery failed" badge on the channel + last error; next transition retries fresh |
| Escalation route send fails | second-route `delay_sec` fires but its channel errors | same backoff as above; escalation retried independently of primary | if still failing, log + badge; recovery still cancels pending escalation | Channels page surfaces failing channel; Test button reproduces |
| Check probe transient error | single tick fails (timeout/reset) | in-tick retry once after 10 s before counting toward `fails_to_down` | counts as one fail only after retry also fails (hysteresis N) | monitor stays `up` until N confirmed fails |
| ICMP capability denied | `pro-bing` CAP error at startup/first ping | auto-select TCP:80 connect probe (§0) | ping monitor still functions via TCP fallback | monitor detail notes "ICMP unavailable — using TCP probe" |
| SQLite busy/locked | WAL contention under load | rely on busy_timeout; result write retried in-tick | drop the single raw sample rather than stall scheduler; rollups self-heal from remaining raws | none (internal) |
