# Implementation Spec — Mock Studio

Read `PLAN.md` first. This file fixes the implementation decisions; do not re-decide them.

## 0. Fixed decisions

| Decision | Value |
|---|---|
| Name | **Mock Studio** (`mockstudio`) · License MIT |
| Language | Go 1.23 single static binary; embedded React UI |
| OpenAPI lib | `pb33f/libopenapi` (3.0/3.1, multi-file refs) + own resolved-route table |
| Port | default 4280 (`--port`); UI at `/__studio/`, API at `/__studio/api/` (double-underscore prefix cannot collide with spec routes; if a spec defines `/__studio` → refuse to start with clear error) |
| State store | SQLite in-memory default; `--state-file path` for persistence; one table per inferred store, JSON column |
| Fake data | own generator honoring `example(s)` > `x-faker` hint (subset: name, email, uuid, url, phone, address, sentence, paragraph, past/future date) > schema synthesis; seeded PCG64 via `--seed` (default 42) |
| Scenario conditions | CEL via `cel-go`; request env: `request.{method,path,params,query,headers,body}` |
| Chunk semantics | chaos `p` evaluated per request with run-scoped RNG (statistically testable); latency uniform min..max |
| Chaos+override precedence | rules first-match-wins top-to-bottom; state ops run only if no rule matched; generated example is final fallback |
| Store inference | RESTful patterns per PLAN §6; override via UI or `x-mockstudio-store: {name, key}` extension |
| ID generation | per key schema: uuid format → uuidv4(seeded), integer → maxid+1, string → nanoid(seeded) |
| Record mode | `--record <baseURL>` proxies unmatched+all routes; `freeze` CLI writes scenario + fixtures dir |
| Response validation | generated/override responses validated against spec response schema; violations → log WARN + `/__studio` badge; `--fail-on-validation-error` exits 1 at startup-time static check + 500s at runtime |

## 1. Repository layout

```
mockstudio/
  cmd/mockstudio/main.go            # serve (default), freeze, version
  internal/
    spec/{load.go, router.go (resolved route table + matcher), validate.go}
    gen/{example.go, faker.go, schema.go (synthesis), deterministic_test.go}
    store/{infer.go, stores.go (CRUD+pagination+filtering), seed.go}
    scenario/{model.go, yaml.go, engine.go (rules eval), sequence.go, chaos.go, cel.go}
    proxy/{record.go, freeze.go, volatile.go (date/id templating detection)}
    server/{handler.go (pipeline), cors.go, log.go (ring 5k + SSE), studioapi.go}
    ui/ (embed)
  web/src/pages/{Routes.tsx, RequestLog.tsx, State.tsx, Scenarios.tsx}
      components/{TryIt.tsx, RouteTree.tsx, LogRow.tsx (replay + pin-as-override buttons),
                  StoreTable.tsx, ScenarioEditor.tsx (YAML w/ monaco), ToggleRail.tsx}
  testdata/specs/{petstore.yaml, github-sub.yaml, stripe-sub.yaml, gnarly1.yaml (allOf/oneOf/discriminator/circular), gnarly2.yaml}
  action/action.yml
  docs/{RECIPES.md (playwright/CI), SCENARIOS.md}
```

## 2. Dependencies

Go: `pb33f/libopenapi`, `google/cel-go`, `modernc.org/sqlite`, `go-chi/chi/v5`, `fsnotify` (spec hot-reload), `matoous/go-nanoid`, `rs/zerolog`. Web: react, monaco (lazy), tanstack-query, tailwind. Dev: vitest (web), `httptest`-based Go suites, playwright (UI E2E).

## 3. Request pipeline (normative, `server/handler.go`)

```
req → CORS (auto preflight from spec if enabled, default on) →
  route match (path template + method; 404 w/ top-3 closest route hints) →
  active scenarios' rules in order (first match wins):
    respond | sequence(next step; resets on scenario re-activation) |
    chaos(error p / latency) — chaos may FALL THROUGH (latency then continue) if no error triggered |
    proxy-through | state-op override
  → else if route maps to store: CRUD handler
      (request-body schema validation → 422 {error, details[]}; pagination: spec params
       page/limit or cursor detected by param names; filtering: exact-match query params matching
       top-level schema fields)
  → else generated example (deterministic seed+route+status)
→ response validation (warn) → ring log entry → SSE to UI
```

## 4. Scenario YAML (frozen format v1 — PLAN §7 plus)

`match`: `{path (template or exact), method?, cel?}`; `respond`: `{status, headers?, body? | bodyExample: <named example>}`; `sequence`: array of respond-objects; `chaos`: `{error?: {status, body?, p}, latency?: {min, max}}`; `state`: `{store: {generate: N} | {fixtures: path}}`; `proxy: true`. Loader validates against JSON schema (`scenario.schema.json`, published); unknown keys = error. UI edits round-trip through the same struct (lossless test).

## 5. Studio API (`/__studio/api`)

`GET /routes` (tree + store mapping + validation badges) · `POST /tryit {method, path, body…}` → executes internally, returns response + which-rule-matched trace · `GET /log?filter=` + `GET /log/stream` (SSE) · `POST /log/{id}/replay` · `POST /log/{id}/pin` → creates override rule in scenario `pinned` (auto-created, active) · `GET/PUT /state/{store}` (rows, edit) · `GET /scenarios`, `PUT /scenarios/{name}` (YAML), `POST /scenarios/{name}/toggle` · `GET /healthz` (also plain `/healthz` unless spec claims it).

## 6. CLI contract

```
mockstudio serve <spec> [--port 4280] [--seed 42] [--scenario file.yaml]... [--state-file f]
  [--record https://real.api] [--quiet] [--fail-on-validation-error] [--no-cors] [--watch]
mockstudio freeze [--out scenarios/recorded.yaml]   # from a --record session's capture db
```
`--watch` hot-reloads spec (stores preserved where schema-compatible, else reset + warn).

## 7. Milestone task lists

**M0** — T1 spec load/resolve + route table + matcher (template params, content-type negotiation minimal: JSON only MVP — decision); T2 schema example generator + determinism test (same seed → byte-equal across runs); T3 serve + 404 hints + terminal request log; T4 corpus tests (5 specs parse+serve; property test: generated bodies validate against own response schema); T5 cold-start bench (300-endpoint < 2 s). *Prism parity.*
**M1** — T1 store inference (+nested scoping, ambiguity list) + CRUD + 422 validation + id gen; T2 pagination + filtering emulation; T3 `--state-file` persistence + restart test; T4 lifecycle E2E per fixture spec (POST→GET→PUT→DELETE→404); T5 `x-mockstudio-store` extension.
**M2** — T1 scenario model + YAML schema + loader; T2 engine: respond/sequence/chaos/state/proxy + precedence tests; T3 CEL env + condition tests; T4 statistical chaos test (10k reqs, ±3%); T5 multi-scenario activation semantics (later-activated wins ties — fixed rule).
**M3** — T1 studio UI: Routes+TryIt (with rule-match trace), RequestLog (SSE, filter, replay); T2 pin-as-override flow (one-flow E2E); T3 State browser/editor; T4 ScenarioEditor (monaco YAML + validate on save + toggle rail); T5 UI perf with 300 routes (virtualized tree).
**M4** — T1 record proxy (capture db: route, req, resp, dedupe by route+status+body-shape hash); T2 freeze (volatile-field detection: ISO dates → `{{now±}}` template, uuids/ids in responses matching request ids → reference templates; imperfect = fine, flag `# TODO volatile?` comments); T3 replay fidelity test; T4 CI polish: exit codes, `--quiet`, healthz, GH Action, RECIPES.md (copy-paste Playwright recipe tested in CI); T5 seed-stability snapshot test for E2E users. Tag v1.0.

## 8. Test mapping

Never-crash-on-weird-spec: corpus grows via bug reports; loader failures must produce actionable errors (assert_golden on messages). Statistical + determinism + lifecycle suites release-blocking. Playwright drives pin-as-override and scenario toggle flows.

## 9. Store inference, validation default & load strategy

**Store inference (`store/infer.go`)** — formalized RESTful heuristic over the resolved route table; ambiguous cases are surfaced, never guessed silently:

```
function inferStores(routes):
  stores = {}
  # collection = GET/POST on /segment ; item = GET/PUT/PATCH/DELETE on /segment/{id}
  for each route:
    (base, idParam) = splitTrailingIdParam(route.path)   # "/pets/{petId}" -> ("/pets", petId)
    name = lastPathSegment(base)                          # "pets"
    if route has {id} tail and method in {GET,PUT,PATCH,DELETE}: stores[name].item += route
    if route ends at base and method in {GET,POST}:          stores[name].collection += route
  for name, s in stores:
    if s has both a collection and an item shape: mark s CONFIRMED, key = the id param name
    else: mark s AMBIGUOUS (partial CRUD) — do NOT auto-create; list in /routes + startup log
  # nested scoping: /users/{userId}/posts -> store "posts" keyed by (userId, postId)
  return CONFIRMED stores; AMBIGUOUS ones fall back to generated examples until user maps them
```

User override (`x-mockstudio-store` or UI) always wins over inference.

**Response-validation default (restating §0/§3 explicitly):** the default is **pass-through** — a response that violates its spec schema is still served to the client, with a `WARN` ring-log entry and a red badge in `/__studio`. Only `--fail-on-validation-error` changes this to a hard error (startup static check exits 1; runtime violations return `500 {error:'validation'}`). Mock Studio never silently drops or mutates a violating response.

**Load strategy:** the spec is fully parsed and the resolved route table built **eagerly at startup** (so the 300-endpoint < 2 s cold-start bench and 404 route hints work); fake-data generation and store tables are **lazy per-request** (first hit to a route synthesizes and caches its generator). `--watch` rebuilds the route table on change, preserving stores where schema-compatible.
