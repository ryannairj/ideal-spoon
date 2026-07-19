# Implementation Spec — Prompt Arena

Read `PLAN.md` first. This file fixes the implementation decisions; do not re-decide them.

## 0. Fixed decisions

| Decision | Value |
|---|---|
| Name | **Prompt Arena** (`arena` on npm as `@prompt-arena/cli`) · License MIT |
| Package | ONE npm package: CLI (commander) + local server (Fastify static+API) + UI (Vite React, prebuilt into `dist-ui/`); Node 22 |
| Project = directory | `arena.yaml` + `prompts/` + `datasets/` + `suites/` + `runs/` (runs gitignored by default via generated `.gitignore` entry, overridable) |
| Template syntax | `{{var}}` handlebars-lite (own 50-line impl: variables + `{{#each}}` NO — variables only; missing var = hard error) |
| Prompt file | `prompts/<name>.md` with YAML frontmatter `{vars: [..], system?, temperature?, max_tokens?}`; body = user template; history auto-snapshots in `prompts/<name>.history/<n>.md` + `manifest.json {parent, message, hash}` |
| Providers | `anthropic`, `openai_compat`, `ollama`; keys ONLY via env var names referenced in arena.yaml (`api_key_env`) |
| Graders (fixed kinds) | `exact`, `regex`, `contains`, `json_schema`, `js` (node:vm, 5 s timeout, no require/fs/net), `llm_rubric` (1–5 + reason; optional `self_consistency: 3` majority), `human` (UI grading queue) |
| Score model | each grader → `{score: 0..1, pass: bool, reason?}`; cell score = weighted mean (suite weights, default equal); case pass = all required graders pass |
| Run artifact | `runs/<iso>-<slug>/manifest.json` + `cells/<case>-<variant>.json` `{request, response, streamRaw?, scores[], usage, cost, latencyMs, promptHash}` |
| Elo | K=32, initial 1000, ties = 0.5; stored per suite in `suites/<name>.elo.json` |
| Concurrency | per-provider `rate: {concurrent: 4, rpm?: n}` in arena.yaml; retries 3 × expo backoff on 429/5xx |
| Baseline | `suites/<name>.yaml` field `baseline: <runId>`; "promote" writes it |
| CI reporters | `--reporter junit|markdown|json`; `--assert` reads suite `thresholds: {min_mean, max_regressions, min_pass_rate}` |
| Ports | UI server 4321 |

## 1. Repository layout (the tool's own repo)

```
prompt-arena/
  src/
    cli/{index.ts, init.ts, run.ts, ui.ts, assert.ts}
    core/{project.ts (load/validate all files), template.ts, snapshot.ts,
          runner/{executor.ts, providers/{anthropic.ts, openaiCompat.ts, ollama.ts}, retry.ts, cost.ts},
          graders/{index.ts, deterministic.ts, jsGrader.ts, llmRubric.ts},
          diffing.ts, elo.ts, reporters/{junit.ts, markdown.ts, json.ts}}
    server/{app.ts, api.ts, sse.ts}
  ui/src/
    pages/{Matrix.tsx, CellInspector.tsx, PromptEditor.tsx, History.tsx, Regression.tsx,
           ArenaAB.tsx, HumanGrade.tsx, CostReport.tsx, SuiteEditor.tsx}
    components/{CellChip.tsx, ScoreBadge.tsx, DiffText.tsx, EloTable.tsx, RunPicker.tsx}
  templates/init/                    # `arena init` scaffold incl. 3 example suites
  fixtures/{recorded/*.json, projects/}
  schemas/{arena.schema.json, suite.schema.json, dataset.schema.json}
```

## 2. Dependencies

`commander, fastify, @fastify/static, zod, yaml, js-yaml NOT (yaml only), papaparse (csv datasets), ajv (json_schema grader + file schemas), diff, nanoid, tiktoken (fallback token est), open` (launch browser). UI: react, monaco (prompt editor), tanstack-query, uplot, tailwind. Dev: vitest, playwright.

## 3. File format contracts (validated with published JSON schemas + friendly errors)

```yaml
# arena.yaml
providers:
  - {name: sonnet, kind: anthropic, model: claude-sonnet-5, api_key_env: ANTHROPIC_API_KEY,
     rate: {concurrent: 4}, price: {inPerM: 3, outPerM: 15}}
  - {name: local, kind: ollama, base_url: http://localhost:11434, model: llama3.3}
grader_model: sonnet
defaults: {temperature: 0}
# datasets/tickets.yaml
cases:
  - {id: t1, vars: {ticket: "..."}, expect: {contains: "refund"}, tags: [billing]}
# suites/triage.yaml
prompt: triage
dataset: tickets
variants: [sonnet, local]            # provider names OR prompt versions "triage@v3"
graders:
  - {kind: json_schema, schema_file: schemas/triage.json, required: true}
  - {kind: llm_rubric, rubric: "Correctly identifies urgency...", weight: 2}
thresholds: {min_mean: 0.8, max_regressions: 0}
baseline: 2026-07-12T…-a1b2
```
`expect` shorthand in cases auto-materializes a deterministic grader for that case only.

## 4. Server API (localhost only)

`GET /api/project` (parsed everything) · `POST /api/run {suite, variants?, onlyFailed?: runId}` → `{runId}`, progress via SSE `/api/runs/{id}/stream` (`cell_done` events) · `GET /api/runs`, `GET /api/runs/{id}` · `POST /api/prompts/{name} {content, message}` (snapshot) · `GET /api/diff?run&baseline` → per-case deltas · `POST /api/ab/start {suite, a, b}` / `POST /api/ab/vote` → elo update · `POST /api/grade {cellRef, score}` (human grader) · `POST /api/baseline {suite, runId}`. UI never writes files except through these (single write path = `core/project.ts`).

## 5. Key behaviors

**Regression view:** join baseline+current cells on case id: improved (fail→pass or +0.1 score), regressed (reverse), new-fail; aggregate deltas; fixture-pair unit test. **Blind A/B:** server strips provider/model fields from cell payloads until vote posted (DOM snapshot test asserts no leak, incl. response headers/usage block); min 10 comparisons before Elo table shows confidence note removal. **Cost:** provider-reported usage preferred; missing → tiktoken estimate flagged `estimated: true`; pricing changes never retro-apply (cost stored at run time).

## 6. Milestone task lists

**M0** — T1 package scaffold + `arena init` (templates incl. example suites) + project loader/validators with friendly errors (path+line via yaml AST); T2 template.ts + tests; T3 executor + anthropic provider + run artifacts; T4 minimal Matrix UI reading runs dir + `arena ui`.
**M1** — T1 openai_compat + ollama + rate/retry (recorded fixtures: stream chunks, malformed, 429 path); T2 deterministic graders + `expect` shorthand; T3 CellInspector (request/rendered prompt/stream/scores); T4 SSE live matrix + only-failed re-run; T5 50×4 virtualization perf.
**M2** — T1 llm_rubric (+self-consistency vote) + js grader sandbox (timeout + no-fs/net escape tests); T2 prompt snapshots + History page + hash-pinned reproducibility (cell → promptHash test); T3 Regression view + baselines + promote.
**M3** — T1 ArenaAB blind flow + elo.ts (unit-tested) + leak-proof snapshot test; T2 HumanGrade queue; T3 cost.ts + CostReport + 5%-reconciliation nightly live smoke; T4 tags/filters in matrix.
**M4** — T1 headless `arena run` + reporters (junit/markdown/json) + `--assert` exit codes; T2 GH Actions recipe (docs + tested workflow in repo); T3 `arena ui --open runs/<id>` for uploaded artifacts; T4 example gallery (3 suites: extraction, summarization-rubric, classification), docs site README. Tag v1.0.

## 7. Test mapping

All runner logic on recorded fixtures (CI hermetic); nightly live smoke across 3 providers. Schema errors golden-tested (message quality is a feature). Playwright: init → run → inspect → regress → A/B happy path. js-grader sandbox escape suite release-blocking.
