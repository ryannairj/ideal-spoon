# Shared Conventions for All Build Plans

These standards apply to every plan in this library unless the plan explicitly overrides them. Individual plans stay focused on *what to build*; this file defines *how we build*.

## Two documents per project

Every project folder contains:

- **`PLAN.md`** — the *what and why*: vision, differentiation, scope, feature specs with acceptance criteria, milestones, risks.
- **`IMPLEMENTATION.md`** — the *exactly how*: fixed decisions (names, ports, versions), full repository layout, exact dependencies, database DDL, API contracts, screen/component inventory, core-algorithm pseudocode, and ordered task lists per milestone.

**Rule for implementing agents:** decisions recorded in `IMPLEMENTATION.md` §0 ("Fixed decisions") and elsewhere in that file are settled — do not re-litigate them. If something proves genuinely impossible (a library is broken, an API changed), choose the nearest equivalent, keep going, and record the deviation in the project repo's `DECISIONS.md`. Anything *not* specified in either document is yours to decide idiomatically — pick the boring option and move on; do not stop to ask.

## How plans are structured

Every `PLAN.md` contains, in order:

1. **Vision** — what the product is and the single job it must do brilliantly.
2. **Why it can win** — differentiation vs. existing tools; the wedge.
3. **Users & use cases** — who it's for and the 3–5 core scenarios.
4. **MVP scope & non-goals** — the hard boundary. Non-goals are binding: do not build them.
5. **Tech stack** — chosen, not surveyed. Agents should use exactly this stack unless something is impossible, in which case document the deviation in `DECISIONS.md` in the project repo.
6. **Architecture** — components and how data flows between them.
7. **Data model** — tables/collections with fields. Types are indicative; adapt idiomatically to the chosen ORM.
8. **Feature specs** — numbered `F1…Fn` with acceptance criteria. Acceptance criteria are testable statements; each one should map to at least one automated test.
9. **Milestones** — `M0…Mn`, each independently shippable, each a checklist an agent can execute top-to-bottom.
10. **Testing strategy** — what to cover beyond the defaults below.
11. **Risks & open questions** — known unknowns; resolve conservatively, note the resolution.

## Engineering defaults

- **Repo layout**: monorepo per project. `apps/` for deployables, `packages/` for shared code (JS/TS), or `cmd/` + `internal/` (Go), or `src/<pkg>` (Python).
- **Language/tooling versions**: TypeScript ≥ 5.x on Node ≥ 22 (use `pnpm`), Go ≥ 1.23, Python ≥ 3.12 (use `uv` + `ruff`), Rust stable (use `cargo clippy`).
- **Formatting/linting**: Prettier + ESLint (flat config) for TS; `gofmt` + `golangci-lint` for Go; `ruff format` + `ruff check` for Python. CI fails on lint errors.
- **Testing**: Vitest (TS), `go test` (Go), `pytest` (Python), Playwright for E2E web flows. Minimum bar: every feature's acceptance criteria covered; CI green before every merge.
- **CI**: GitHub Actions. One workflow: lint → typecheck → unit tests → build → E2E (if applicable). Cache dependencies.
- **Config**: 12-factor. All secrets via env vars; ship a `.env.example`. Never commit secrets.
- **Database**: default to SQLite for single-node/self-hosted tools (via Drizzle/`sqlc`/SQLAlchemy), Postgres for multi-tenant SaaS. Migrations are versioned and committed from day one.
- **Auth (when needed)**: email magic link + OAuth (Google, GitHub) via a library (Auth.js / Lucia-style session tables). Never hand-roll password hashing beyond `argon2id` via a maintained lib.
- **LLM usage (when the plan calls for it)**: put all model calls behind a single `llm/` module with a provider interface, so models are swappable. Log token usage. Fail gracefully when the provider is down — the app must degrade, not crash.
- **Observability**: structured logs (JSON) with levels; a `/healthz` endpoint on every server.
- **Docs**: every project ships `README.md` (setup in ≤ 5 commands), `ARCHITECTURE.md` (kept current), and `DECISIONS.md` (append-only log of deviations from the plan).
- **Licensing**: MIT by default for tools intended as open source; plans flag exceptions.

## Specification rigor (for AI-agent executability)

These standards close the gaps that most often force an implementing agent to invent a decision. They apply to every plan.

### Pseudocode for non-trivial algorithms

Any algorithm that is more than a single obvious step must appear as pseudocode in `IMPLEMENTATION.md`, not only as prose. This includes clustering, similarity/scoring, decay, ranking, diffing, merging, conflict resolution, and any multi-branch heuristic. Rule of thumb: if the prose contains a name like "kmeans-lite", "trigram similarity", "decay with half-life", or "LWW tie-break", there must be 5–15 lines of pseudocode showing the actual computation, inputs, outputs, and boundary behavior.

### Tuning constants marked `@TUNE`

Every empirically-chosen constant (similarity thresholds, decay half-lives, Elo K-factors, overlap tolerances, cosine cutoffs, EMA α, retry counts) must be either:

- **Justified** — a one-line rationale or reference next to the value, or
- **Marked `@TUNE`** — written as `@TUNE(name=value)` in `IMPLEMENTATION.md` §0, with a note stating the milestone that calibrates it against a named fixture set. `@TUNE` means "this is a starting value, not a settled decision; the calibration task owns final tuning."

Never lock an unexplained magic number as if it were settled.

### Error Recovery & Graceful Degradation section

Every `IMPLEMENTATION.md` that calls an external service (network, LLM, filesystem watchers, OS APIs) must contain a section titled **"Error Recovery & Graceful Degradation"** that specifies, for each failure class:

- **Trigger** — the exact condition that counts as failure (timeout ms, HTTP status class, malformed/non-schema output, exception type).
- **Backoff** — retry count, base delay, cap, and jitter (state "no retry" explicitly when that is the choice).
- **Fallback** — the degraded result the user/system gets when retries are exhausted (cached value, deterministic substitute, skip-with-warning, queue-for-later).
- **User-facing UX** — what the user sees (banner, toast, silent log, disabled control).

"Retry gracefully" with no trigger/backoff/fallback is not acceptable.

### LLM prompt templates as versioned fixtures

Projects that call an LLM must commit representative prompt templates as versioned files under `fixtures/prompts/` (e.g. `fixtures/prompts/<task>.md`), referenced by path from `IMPLEMENTATION.md`. Each template pins its role/system message, the input variables (as `{{placeholders}}`), the required output schema, and one worked example. This removes prompt invention at implementation time. Prompt changes are reviewed like code and versioned (a `# v1`, `# v2` header suffices for the MVP).

### Boundary conditions as acceptance criteria

Boundary and edge-case behavior must be expressed as explicit, testable acceptance criteria in `PLAN.md` feature specs — not left to idiom. Examples: bucket-edge assignment ("a result whose timestamp equals a rollup boundary belongs to the newer bucket"), overlapping spans, concurrent-merge tie-breaks, empty/one-element inputs, and platform-specific limits. Each such AC must map to at least one automated test.

## Definition of done (per milestone)

- All checklist items complete; acceptance criteria for the milestone's features pass via automated tests.
- CI green; lint/typecheck clean.
- README updated so a newcomer can run the milestone's functionality locally.
- No TODOs left in code without a linked issue/note in `DECISIONS.md`.

## Clone etiquette

Several plans are clones of commercial products. Clones here mean *re-implementations of the concept*: never copy proprietary assets, branding, UI artwork, or decompiled code. Names in plan folders (e.g. `termius-clone`) are working titles — ship under an original name.
