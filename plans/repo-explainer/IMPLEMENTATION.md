# Implementation Spec — Repo Explainer

Read `PLAN.md` first. This file fixes the implementation decisions; do not re-decide them.

## 0. Fixed decisions

| Decision | Value |
|---|---|
| Product name | **Atlaslens** |
| License | AGPL-3.0 (self-hostable, protects hosted version) |
| Stack | Next.js 15 (app router) + TS 5, Node 22, pnpm; Postgres 16 + pgvector via Drizzle; Redis 7 + BullMQ |
| Monorepo | pnpm workspaces: `apps/web`, `apps/worker`, `packages/{db,analysis,llm,shared}` |
| Parsing | `web-tree-sitter` (WASM grammars vendored in `packages/analysis/grammars/`): ts/tsx, javascript, python, go, rust, java, ruby, c_sharp |
| LLM defaults | bulk model `deepseek-chat`, synthesis model `claude-sonnet-5`, embeddings `text-embedding-3-small` (1536 d) — all via `packages/llm` provider registry, env-overridable |
| Token budget | default 4M input tokens per analysis (`ANALYSIS_TOKEN_BUDGET`); over-budget → sampled mode |
| Repo size cap | clone depth 1; > 500k LOC → sampled mode with coverage % banner |
| Queue jobs | one BullMQ queue `analysis`, job = one pipeline stage (chained), checkpointed in DB |
| Diagram | Mermaid `graph TD`, ≤ 14 module nodes, leftover cluster "other" |
| Ports | web 3000, worker healthz 3001 |

## 1. Repository layout

```
atlaslens/
  apps/web/src/
    app/
      page.tsx                        # submit form + recent repos
      r/[repoSlug]/page.tsx           # overview (atlas home)
      r/[repoSlug]/modules/[moduleId]/page.tsx
      r/[repoSlug]/paths/[pathId]/page.tsx     # reading path stepper
      r/[repoSlug]/file/[...path]/page.tsx     # annotated file viewer
      r/[repoSlug]/chat/page.tsx
      api/repos/route.ts              # POST submit
      api/repos/[id]/{status,refresh}/route.ts
      api/chat/route.ts               # POST question (streams SSE)
    components/{ProgressCard, MermaidView, ModuleCard, ReadingStepper,
                CodePane (shiki), CitationLink, ChatPanel, StatsBadges}
  apps/worker/src/
    index.ts                          # BullMQ workers + healthz
    stages/{s1_inventory.ts, s2_parse.ts, s3_rank.ts, s4_summarize.ts,
            s5_synthesize.ts, s6_index.ts}
    lib/{git.ts, budget.ts, checkpoints.ts}
  packages/db/src/{schema.ts, client.ts, migrations/}
  packages/analysis/src/{treesitter.ts, symbols.ts, imports.ts, entrypoints.ts,
                         frameworks/registry.ts, frameworks/{next.ts,django.ts,rails.ts,
                         spring.ts,express.ts,fastapi.ts,gin.ts,laravel.ts}, pagerank.ts, churn.ts}
  packages/llm/src/{provider.ts, anthropic.ts, openai_compat.ts, embeddings.ts,
                    prompts/{file_summary.ts, module_rollup.ts, architecture.ts,
                             reading_paths.ts, glossary.ts, qa.ts}, citations.ts}
  packages/shared/src/types.ts        # zod schemas shared web/worker
  fixtures/golden-repos/{express-app.tar.gz, django-app.tar.gz, go-cli.tar.gz}
  docker-compose.yml                  # pg+pgvector, redis, web, worker
```

## 2. Dependencies

Web: `next@15`, `react@18`, `drizzle-orm`, `postgres`, `bullmq`, `ioredis`, `zod`, `shiki`, `mermaid`, `tailwindcss@4`, `ai` (SSE streaming helper) — no auth lib in MVP (public tool; rate-limit by IP via `@upstash/ratelimit`-style middleware, self-rolled on Redis).
Worker adds: `web-tree-sitter`, `simple-git`, `tiktoken` (budgeting), `p-limit`.

## 3. Configuration (`.env.example`)

```
DATABASE_URL=postgres://…  REDIS_URL=redis://…
LLM_BULK_PROVIDER=deepseek  LLM_BULK_MODEL=deepseek-chat  DEEPSEEK_API_KEY=
LLM_SYNTH_PROVIDER=anthropic LLM_SYNTH_MODEL=claude-sonnet-5 ANTHROPIC_API_KEY=
EMBEDDINGS_PROVIDER=openai EMBEDDINGS_MODEL=text-embedding-3-small OPENAI_API_KEY=
ANALYSIS_TOKEN_BUDGET=4000000  CLONE_DIR=/tmp/atlaslens  MAX_REPO_MB=500
```

## 4. Database schema (Drizzle — shown as SQL)

```sql
CREATE TABLE repos (id uuid PK default gen_random_uuid(), url text UNIQUE NOT NULL,
  slug text UNIQUE NOT NULL, default_branch text, head_sha text,
  status text NOT NULL DEFAULT 'queued' CHECK (status IN ('queued','analyzing','ready','failed')),
  stats jsonb NOT NULL DEFAULT '{}', sampled boolean DEFAULT false, coverage_pct int,
  created_at timestamptz DEFAULT now());
CREATE TABLE analyses (id uuid PK, repo_id uuid REFERENCES repos NOT NULL, head_sha text NOT NULL,
  stage text NOT NULL DEFAULT 's1', stage_state jsonb NOT NULL DEFAULT '{}',   -- checkpoints
  cost_tokens_in bigint DEFAULT 0, cost_tokens_out bigint DEFAULT 0,
  started_at timestamptz, finished_at timestamptz, error text);
CREATE TABLE files (id uuid PK, repo_id uuid NOT NULL, path text NOT NULL,
  blob_sha text NOT NULL, language text, loc int, churn int, rank real,
  summary text, symbols jsonb, UNIQUE(repo_id, path));
CREATE TABLE modules (id uuid PK, repo_id uuid NOT NULL, name text, path_prefixes jsonb,
  responsibility text, narrative text, diagram_mermaid text, sort int);
CREATE TABLE reading_paths (id uuid PK, repo_id uuid NOT NULL, title text, description text,
  steps jsonb NOT NULL);  -- [{path, startLine, endLine, commentary}]
CREATE TABLE chunks (id uuid PK, repo_id uuid NOT NULL, path text, start_line int, end_line int,
  kind text CHECK (kind IN ('code','summary','doc')), content text,
  embedding vector(1536));
CREATE INDEX chunks_vec ON chunks USING hnsw (embedding vector_cosine_ops);
CREATE INDEX chunks_fts ON chunks USING gin (to_tsvector('english', content));
CREATE TABLE qa_threads (id uuid PK, repo_id uuid, messages jsonb, created_at timestamptz);
```

## 5. API contract

| Route | Behavior |
|---|---|
| `POST /api/repos` `{url}` | validates GitHub URL, dedupes by url+head, enqueues s1 → `{repoId, slug}` |
| `GET /api/repos/{id}/status` | `{status, stage, pct, tokensSpent, error?}` (poll 2 s from ProgressCard) |
| `POST /api/repos/{id}/refresh` | re-clone; unchanged blob_sha files keep summaries (incremental) |
| `POST /api/chat` `{repoId, threadId?, question}` | SSE stream `{delta}` then `{citations:[{path,startLine,endLine}]}` |
| Pages read via server components with Drizzle directly (no extra API). | |

## 6. Pipeline stage contracts (worker)

- **s1_inventory** → writes `repos.stats {files, loc, languages{}, buildFiles[], hasCI, docs[]}`; harvests README/docs into `chunks(kind='doc')` raw.
- **s2_parse** → per supported file: symbols `[{name, kind, startLine, endLine, exported}]`, import edges; entry points via `frameworks/registry` (each framework module exports `detect(tree, inventory)` + `entryPoints()` + `routeMap()?`). Unknown frameworks: heuristic mains (`main()`, `bin` entries, `server.listen`).
- **s3_rank** → PageRank (damping 0.85, 30 iters) over import graph × `1 + log(1+churn)`; picks full-file set under budget: rank order, file token cost via tiktoken; rest get signatures-only treatment.
- **s4_summarize** → bulk model; file prompt returns strict JSON `{purpose, keyItems[], notable}` (zod-validated, 1 retry on parse fail); directory rollups bottom-up.
- **s5_synthesize** → synthesis model: modules (cluster by dir + import cohesion; must cover ≥ 90% of ranked files), architecture narrative, mermaid, 3–5 reading paths, glossary. **Citation validator** (`packages/llm/citations.ts`): every `{path, startLine, endLine}` must exist in `files` and be within line count; invalid → one re-prompt with error list; still invalid → drop claim, log.
- **s6_index** → chunk code (per symbol, max 120 lines) + summaries → embeddings (batch 128).

Each stage: idempotent, writes checkpoint to `analyses.stage_state`, re-runnable from failure.

## 7. Q&A retrieval

Hybrid: top 12 by cosine + top 12 by FTS rank → RRF merge → top 8 → prompt with instruction to cite paths + refuse if insufficient. Confidence gate: if best RRF score < threshold (tuned on eval set), answer template "Not found in this repo's index."

## 8. Milestone task lists

**M0** — T1 monorepo scaffold + compose + migrations; T2 submit form → repos row → clone in worker (s1) with progress; T3 status polling UI; T4 CI (lint, typecheck, vitest, golden-repo tarball fixtures unpack).
**M1** — T1 tree-sitter WASM loading + symbols/imports; T2 framework registry + 8 detectors; T3 pagerank + churn (git log --numstat); T4 raw map page (file tree + ranked list + import table) — ships deterministic value; T5 stage checkpointing + resume test.
**M2** — T1 llm provider layer + budgeter; T2 s4 summaries (recorded-fixture tests); T3 s5 synthesis + citation validator (+ its unit suite); T4 overview page (StatsBadges, MermaidView with node cap/cluster, module links); T5 module pages.
**M3** — T1 reading path stepper + CodePane (shiki, highlighted ranges) + localStorage progress; T2 annotated file viewer with citation deep links (`#L40-L60`); T3 refresh-incremental (blob_sha diff, re-synthesize only if >5% files changed else patch summaries); T4 token accounting surfaced in status.
**M4** — T1 s6 embeddings + hybrid retrieval + chat SSE UI; T2 eval harness (`fixtures/eval/qa.yaml`, 20 Q/A over golden repos; script computes correct-with-citation %; CI nightly live, PR-time recorded); T3 share/OG images (satori) + public atlas routes; T4 rate limiting + docker docs. Tag v1.0.

## 9. Test mapping

Golden-repo snapshot tests per deterministic stage (s1–s3) — snapshots committed. Citation validator: out-of-range/renamed/deleted cases. Recorded-LLM fixtures (JSON transcripts in `fixtures/llm/`) for s4/s5/qa unit tests; nightly workflow `eval.yml` runs live and posts metrics to job summary. Playwright: submit → ready → click module → citation → file viewer.
