# Shared Conventions for All Build Plans

These standards apply to every plan in this library unless the plan explicitly overrides them. Individual plans stay focused on *what to build*; this file defines *how we build*.

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

## Definition of done (per milestone)

- All checklist items complete; acceptance criteria for the milestone's features pass via automated tests.
- CI green; lint/typecheck clean.
- README updated so a newcomer can run the milestone's functionality locally.
- No TODOs left in code without a linked issue/note in `DECISIONS.md`.

## Clone etiquette

Several plans are clones of commercial products. Clones here mean *re-implementations of the concept*: never copy proprietary assets, branding, UI artwork, or decompiled code. Names in plan folders (e.g. `termius-clone`) are working titles — ship under an original name.
