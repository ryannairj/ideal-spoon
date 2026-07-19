# Mock Studio — instant, stateful API mocks from an OpenAPI spec

**Category:** Dev tool · **Difficulty:** Medium · **Platform:** CLI + local web UI, single binary

## 1. Vision

`mockstudio serve openapi.yaml` → a running mock of the whole API in 2 seconds: schema-valid example responses, a web UI to sculpt scenarios ("this user 404s", "orders endpoint is slow and flaky today"), **stateful CRUD** that actually remembers what you POST, and request logs that double as a debugging console. Frontend teams stop waiting for backends; E2E suites get deterministic worlds.

## 2. Why it can win

- Prism gives static examples; WireMock is Java + config sprawl; Mockoon is GUI-first without spec fidelity. The gap: **spec-faithful + stateful + scenario-scriptable** in one no-dependency binary.
- Scenarios-as-files (committable, shareable, CI-loadable) make mock setups reviewable artifacts, not tribal knowledge.

## 3. Users & use cases

Frontend devs, mobile devs, QA engineers, demo-givers.

1. Serve a spec; the app under development points at `localhost:4280`; every endpoint answers validly.
2. Seed 50 fake users via the UI (schema-aware faker) → GET/POST/PUT/DELETE behave consistently against that state.
3. Scenario "checkout-degraded": payments endpoint returns 503 20% of the time with 2 s latency → toggle on, watch the frontend's retry UX.
4. CI: `mockstudio serve spec.yaml --scenario fixtures/e2e.yaml --port 4280` behind Playwright.
5. Record mode: proxy the real staging API once, save responses as a scenario.

## 4. MVP scope & non-goals

**MVP:** OpenAPI 3.0/3.1 (JSON/YAML, multi-file refs); dynamic example generation; stateful resource stores inferred from RESTful path patterns (+ manual mapping); scenario files (overrides, sequences, latency/error injection, proportional chaos); web UI (routes, state browser, request log, scenario editor); record-from-proxy; CORS/preflight handling; single binary.
**Non-goals:** GraphQL/gRPC (adapter seam noted, later), cloud/hosted mode, contract *testing* (we're the server side only), auth simulation beyond static token checks, spec editing (BYO spec).

## 5. Tech stack

- **Core:** Go 1.23 — router built from the spec (`pb33f/libopenapi` for parsing/validation), embedded React UI, SQLite in-memory (file-backed optional) for state stores, single static binary.
- **Fake data:** schema-driven generator honoring formats/enums/patterns/min-max + `x-faker` extension hints; seeded (deterministic with `--seed`).
- **Scenario engine:** YAML rules evaluated per request (match → behavior); tiny CEL expression support for conditions (`request.query.userId == "13"`).

## 6. Architecture & request pipeline

```
request → route match (spec) → scenario rules (first match wins):
   override (fixed body/status/headers) | sequence (1st call → A, 2nd → B, …)
   | chaos (p% error, latency dist) | proxy-through | state op
 → else stateful store (if resource-mapped): CRUD against SQLite w/ schema validation of request bodies (422 on violation, like a real API)
 → else generated example (deterministic per seed+path)
response → request log ring buffer (+SSE to UI)
```

State inference: `GET/POST /things`, `GET/PUT/PATCH/DELETE /things/{id}` → `things` store keyed by `{id}` (id generated per schema type on POST). Nested (`/users/{uid}/orders`) become scoped stores. Ambiguous cases listed in UI for one-click manual mapping.

## 7. Scenario file format (committable)

```yaml
name: checkout-degraded
seed: 42
state:
  users: {generate: 50}
  orders: {fixtures: ./orders.json}
rules:
  - match: {path: /payments, method: POST}
    chaos: {error: {status: 503, p: 0.2}, latency: {min: 800ms, max: 2s}}
  - match: {path: /users/13}
    respond: {status: 404, body: {error: "user suspended"}}
  - match: {path: /exports, method: POST}
    sequence: [{status: 202}, {status: 200, bodyExample: done}]
```

## 8. Feature specs

**F1 — Serve.** Multi-file spec resolution; hot-reload on spec change; unmatched routes → 404 with "closest routes" hint; response validation warnings (generated/override responses checked against spec, mismatches logged loudly). *AC:* petstore + 3 gnarly real-world specs (allOf/oneOf/discriminators, circular refs) serve correctly; cold start < 2 s for a 300-endpoint spec.

**F2 — Example generation.** Priority: spec `example(s)` > `x-faker` hints > schema-driven synthesis; deterministic under `--seed`; arrays sized 2–5; dates recent; enums rotated. *AC:* generated bodies validate against their own response schema 100% on the fixture corpus (property test).

**F3 — Stateful stores.** As §6: full CRUD semantics, request-body validation, pagination emulation (`page/limit` or `cursor` per spec params), filtering on exact-match query params that correspond to schema fields. *AC:* POST→GET(id)→PUT→GET→DELETE→404 flow works on inferred stores of the fixture specs; restart with `--state-file` preserves state.

**F4 — Scenario engine.** File + UI editing (UI writes the same YAML); toggle scenarios live; sequences reset per scenario activation; CEL conditions. *AC:* rules apply in order with first-match; chaos probabilities within ±3% over 10k requests (statistical test); scenario YAML round-trips through UI edits losslessly.

**F5 — Web UI.** Routes tree with try-it panel, live request log (filter, replay a request, "pin as override" from any logged pair), state browser (table view per store, edit rows), scenario toggles/editor. *AC:* log→pin-as-override→replayed request returns pinned response, in one flow; UI fully functional with 300 routes.

**F6 — Record mode.** `--record https://staging.api.com` proxies through, captures request/response pairs, dedupes by (route, status, body-shape), then `mockstudio freeze` writes a scenario + fixtures. *AC:* recorded scenario replays byte-similar bodies (volatile fields — dates/ids — templated automatically where detected).

**F7 — CI ergonomics.** `--port`, `--scenario`, `--seed`, `--quiet`, health endpoint, `--fail-on-validation-error`; GitHub Action wrapper. *AC:* documented Playwright recipe works copy-paste; deterministic seed makes E2E snapshots stable across runs.

## 9. Milestones

- **M0 (1):** parse+route+static example generation, CLI serve, request log (terminal). *Ships: Prism parity.*
- **M1 (2):** deterministic generator + stateful stores + validation-as-a-real-API. *Ships: the differentiator.*
- **M2 (3):** scenario engine + YAML format + chaos/sequences.
- **M3 (4):** web UI (routes, log, state, scenarios, pin-as-override).
- **M4 (5):** record/freeze mode, CI polish, Action, docs + gnarly-spec corpus hardening. *Ships: v1.0.*

## 10. Testing

- Spec corpus (petstore, GitHub, Stripe subset, 2 pathological hand-made) parsed/served in CI; generated-example schema-validity property test.
- Statistical chaos test; sequence/state machine table tests.
- Playwright E2E driving the UI; golden scenario round-trip tests.

## 11. Risks & open questions

- OpenAPI edge-case swamp (discriminators, webhooks, callbacks) → serve what's solid, log-and-skip what isn't, corpus grows with bug reports; never crash on a weird spec.
- State inference wrong for non-RESTful specs → always overridable in UI + `x-mockstudio-store` extension for spec authors.
- Scope creep toward API gateway features → non-goals list is the fence; single-binary constraint is the forcing function.
