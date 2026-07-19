# Prompt Arena — local-first playground for comparing prompts and models like an engineer

**Category:** Dev tool / AI · **Difficulty:** Medium · **Platform:** Local web app (single command), files-on-disk project format

## 1. Vision

Prompt engineering today is vibes in a chat window. Prompt Arena is the missing lab bench: define a prompt with variables, a dataset of test cases, and graders; run it across N models/providers side by side; see a scored matrix, diffs between runs, and cost/latency — then iterate the prompt and *know* whether v7 beat v6. Everything is files in your repo (promptfoo-style but with a real UI), so prompt work gets code review like code.

## 2. Why it can win

- Cloud eval platforms (Braintrust, LangSmith) want your data and a subscription; promptfoo is CLI-first with a thin UI. The gap: **local-first + great comparison UX + files-as-truth** — runs in your repo, secrets stay in your env, diffs live in git.
- The side-by-side "arena" view (same case, 4 models, human A/B pick) generates preference data teams actually use to choose a default model.

## 3. Users & use cases

Developers building LLM features; teams choosing between models; prompt maintainers preventing regressions.

1. `arena init && arena ui` → create prompt "support-triage" with `{{ticket}}` variable, paste 20 real tickets as cases.
2. Run against Claude/GPT/DeepSeek/local-Ollama → matrix: grader scores, cost, latency per cell; click any cell for the full transcript.
3. Edit prompt → re-run → regression view: which cases got better/worse vs baseline run.
4. Blind A/B: arena hides model names, you pick winners, it reveals + tallies Elo.
5. CI: `arena run suite.yaml --assert` fails the build if score drops below threshold.

## 4. MVP scope & non-goals

**MVP:** project file format; providers (Anthropic, OpenAI-compatible, Ollama); prompt versions (auto-snapshotted); datasets (inline YAML/CSV/JSONL import); graders: deterministic (exact/regex/contains/JSON-schema/custom JS) + LLM-rubric grader + human grading UI; run matrix + cell inspector + diff/regression views; blind A/B with Elo; cost/latency capture; CLI runner + CI assert mode.
**Non-goals:** hosted/multi-user collaboration (files + git *are* the collaboration), tracing/observability of production apps, agent/multi-turn tool loops in MVP (single-turn + system prompts first; multi-turn scripted conversations v1.1), fine-tuning.

## 5. Tech stack

- **App:** one npm package: Node 22 CLI (`arena`) that serves a local Vite React UI; no database — **project = directory** (`arena.yaml`, `prompts/*.md`, `datasets/*.{yaml,csv,jsonl}`, `runs/*.json` gitignore-optional).
- **Runner:** provider-agnostic client with streaming, concurrency limits, retries/backoff, seeded sampling params; exact request/response persisted per cell.
- **Custom JS graders:** run in `node:vm` with timeout; LLM graders use a designated grader-model config.

## 6. Project format (files are the product)

```
arena.yaml            # providers (env-var keys), models, defaults, grader model
prompts/triage.md     # frontmatter: vars, temperature, system; body = template (handlebars-lite)
prompts/triage.history/  # auto-snapshots v1..vN with message + parent
datasets/tickets.yaml # cases: [{vars: {ticket: "..."}, expect: {...}, tags: [...]}]
suites/triage.yaml    # prompt × dataset × models × graders + assert thresholds
runs/2026-07-19T.../  # manifest.json + cells/*.json (request, response, scores, usage)
```

## 7. Feature specs

**F1 — Prompt editing + versioning.** Markdown editor with variable highlighting/preview against a sample case; every run pins an auto-snapshot version; history browser with diffs. *AC:* runs are always reproducible-attributable (cell → exact prompt text hash); template rendering covered by unit tests (missing var = hard error, not silent blank).

**F2 — Run matrix.** Grid: rows = cases, columns = model (or prompt-version) variants; cell = pass/fail + score chips + cost + ms; sticky headers; filter by tag/failures; aggregate row (mean score, total cost, p95 latency). Cell inspector: rendered prompt, raw request, streamed response, grader breakdowns. *AC:* 50 cases × 4 models renders smoothly (virtualized); re-run only-failed-cells supported; concurrent execution respects per-provider rate config.

**F3 — Graders.** Chainable per suite: deterministic set; JSON-schema validity; custom JS `(response, testCase) => {score, reason}`; LLM rubric grader (rubric text → 1–5 + reason, grader model configurable, self-consistency n=3 vote option). *AC:* grader unit fixtures; LLM-grader results carry reasons; JS grader timeout/sandbox enforced (no fs/net — test).

**F4 — Diff & regression.** Pick baseline run → current run: per-case delta (improved/regressed/unchanged/new-fail), response text diff view, aggregate movement. *AC:* regression view flags exactly the changed cells on a fixture pair; "promote as baseline" persists to suite file.

**F5 — Blind A/B + Elo.** Arena mode: same case, two anonymized responses, pick better / tie / both bad; Elo table per suite with confidence note (min comparisons). *AC:* model identity leak-proof in UI until reveal (snapshot test on DOM); Elo math unit-tested.

**F6 — CLI + CI.** `arena run suites/triage.yaml` (headless, junit/markdown reporters), `--assert` reads thresholds from suite (`min_mean: 0.8`, `max_regressions: 0`); exit codes accordingly; env-only secrets. *AC:* documented GitHub Actions recipe works copy-paste; run artifacts uploadable and re-openable in the UI (`arena ui --open runs/<id>`).

**F7 — Cost/latency accounting.** Token usage from provider responses × pricing table (editable, versioned in arena.yaml); per-run totals; "cost per solved case" metric. *AC:* totals within 5% of provider dashboards on live smoke; pricing table changes never mutate historical runs.

## 8. Milestones

- **M0 (1):** project format + CLI runner (one provider) + JSON run artifacts + minimal matrix UI. 
- **M1 (2):** all providers, concurrency/retries, deterministic graders, cell inspector. *Ships: useful bench.*
- **M2 (3):** LLM + JS graders, prompt versioning/history, diff/regression + baselines. *Ships: the iteration loop.*
- **M3 (4):** blind A/B + Elo, cost accounting, tags/filters. 
- **M4 (5):** CI assert mode + reporters, docs, example gallery (3 ready-made suites). *Ships: v1.0.*

## 9. Testing

- Recorded-provider fixtures for all runner logic (streaming chunks, malformed responses, 429 retry paths); nightly live smoke across providers.
- Golden run artifacts → UI snapshot tests; matrix virtualization perf test.
- File-format round-trip + schema validation (arena.yaml, suites, datasets) with helpful error messages tested.

## 10. Risks & open questions

- Provider API drift → thin adapter per provider + contract fixtures; failures degrade per-cell, never kill the run.
- LLM-grader trustworthiness → always show reasons, offer self-consistency voting, and keep deterministic graders first-class so teams can mix.
- Scope gravity toward "observability platform" → the fence: no production tracing, files stay the source of truth.
