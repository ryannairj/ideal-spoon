# Repo Explainer — paste a GitHub repo, get a guided tour

**Category:** Dev tool · **Difficulty:** Medium · **Platform:** Web app (self-hostable)

## 1. Vision

Point it at any GitHub repo and get what a senior engineer would give you on day one: what this codebase does, how it's laid out, where the entry points are, how data flows, and a set of guided "reading paths" ("how a request becomes a response", "how auth works here"). Onboarding to an unfamiliar codebase drops from days to an hour.

## 2. Why it can win

- GitHub's own AI answers questions but doesn't *teach structure*. Existing "chat with repo" tools are RAG over files with no map. Our output is a **persistent, browsable atlas** (diagrams + narratives + linked source ranges), generated once, refreshed on demand — not an ephemeral chat.
- Deterministic analysis first (tree-sitter symbols, imports, entry points), LLM narration second — so explanations cite real files/lines and hallucinate less.

## 3. Users & use cases

New team members, open-source contributors, engineers evaluating a dependency, tech leads doing due diligence.

1. "Explain this repo" → overview page in ~2 minutes for a mid-size repo.
2. Click a module in the architecture diagram → its role, key files, key symbols.
3. Follow the "trace a request" reading path file-by-file with commentary.
4. Ask a free-form question; answer cites `file:line` links into the atlas.

## 4. MVP scope & non-goals

**MVP:** public GitHub repos (git clone by URL); analysis pipeline; atlas UI (overview, module map, reading paths, file annotations); Q&A chat grounded in the index; refresh on new commits.
**Non-goals:** private-repo OAuth (v1.1), IDE plugins, PR review, editing code, monorepo-of-monorepos scale (> ~500k LOC gets sampled with a visible notice).

## 5. Tech stack

- **App:** Next.js 15 + TypeScript, Postgres (Drizzle) + pgvector, BullMQ + Redis for the pipeline.
- **Analysis worker:** Node + tree-sitter (grammars: TS/JS, Python, Go, Rust, Java, Ruby, C#) for symbols/imports; `git` CLI for clone/log; simple heuristics registry for frameworks (Next, Django, Rails, Spring, etc.).
- **LLM:** provider-agnostic `llm/` module (per CONVENTIONS): cheap model (e.g. DeepSeek) for bulk file summaries, strong model for synthesis passes. Embeddings for Q&A retrieval.
- **Diagrams:** generate Mermaid; render client-side.

## 6. Architecture & pipeline

```
URL → clone → inventory → parse (tree-sitter) → rank → summarize (LLM, bottom-up)
    → synthesize (modules → architecture → reading paths) → index (pgvector) → atlas
```

Pipeline stages (each idempotent, checkpointed in DB so retries resume):

1. **Inventory:** file tree, languages, LOC, build files, CI, README/docs harvest.
2. **Parse:** symbols (functions/classes/exports), import graph, entry points (main/bin/server bootstraps/route registrations via framework heuristics).
3. **Rank:** PageRank over import graph + git-churn weighting → "important files" list; token budgeter picks what gets full-file LLM treatment vs signature-only.
4. **Summarize:** per-file summaries (map), per-directory/module rollups (reduce).
5. **Synthesize:** architecture narrative, module responsibilities, data-flow description, 3–5 reading paths, glossary of domain terms. Every claim must carry `citations: [{path, startLine, endLine}]`; a validator rejects outputs citing nonexistent ranges and re-prompts.
6. **Index:** chunk code+summaries → embeddings for chat.

## 7. Data model

- `repos(id, url, default_branch, head_sha, status{queued,analyzing,ready,failed}, stats_json, created_at)`
- `analyses(id, repo_id, head_sha, pipeline_state_json, cost_tokens, finished_at)`
- `files(id, repo_id, path, language, loc, churn, rank, summary, symbols_json)`
- `modules(id, repo_id, name, path_prefixes_json, responsibility, narrative, diagram_mermaid)`
- `reading_paths(id, repo_id, title, description, steps_json /*[{path, lines, commentary}]*/)`
- `chunks(id, repo_id, path, start_line, end_line, content, embedding vector)`
- `qa_threads(id, repo_id, messages_json)`

## 8. Feature specs

**F1 — Submit & analyze.** Paste URL → job with live progress (stage + % + token spend). *AC:* a 50k-LOC repo completes < 5 min with cheap-model bulk pricing; failure at any stage is resumable, not restart-from-zero.

**F2 — Overview page.** What/why/how summary, tech stack badges, stats, architecture Mermaid diagram (modules + arrows = import weight), "start here" pointers. *AC:* every module in the diagram links to its module page; diagram renders for repos with 1–30 modules without overlap soup (cap nodes, group leftovers as "other").

**F3 — Module pages.** Responsibility narrative, key files ranked, key symbols with one-liners, inbound/outbound dependency lists. *AC:* all cited file:line links open the annotated file view at that range.

**F4 — Reading paths.** Ordered steps: file pane (highlighted range) + commentary pane, next/prev navigation. Auto-generated set must include an "entry point → core flow" path. *AC:* steps always reference real ranges (validator-enforced); user can check off progress (localStorage).

**F5 — Grounded Q&A.** Chat retrieves from chunks (hybrid: vector + keyword) and answers with citations; refuses ("not found in this repo") rather than guessing when retrieval is weak. *AC:* eval set of 20 Q/A pairs on 3 known repos ≥ 80% correct-with-citation, 0 fabricated paths (automated check that cited paths exist).

**F6 — Refresh.** "Update to latest" re-runs incrementally: unchanged files (same blob sha) keep summaries; only changed subtree re-summarized/synthesized. *AC:* a 5-file change re-analyzes in < 1/10 of full-run tokens.

**F7 — Share.** Public atlas URL per analyzed repo (read-only), OG image with repo name + diagram thumbnail.

## 9. Milestones

- **M0 (1):** Next.js + Postgres + queue scaffold; clone + inventory stage; progress UI. 
- **M1 (2):** tree-sitter parsing, import graph, ranking; raw "map" page (no LLM yet — ship deterministic value first).
- **M2 (3):** summarize + synthesize passes with citation validator; overview + module pages. *Ships: MVP demo.*
- **M3 (4):** reading paths + annotated file viewer; refresh-incremental. 
- **M4 (5):** embeddings + grounded Q&A + eval harness; share pages; Docker compose for self-hosting. *Ships: v1.0.*

## 10. Testing

- Golden-repo fixtures (small Express app, Django app, Go CLI) committed as tarballs; pipeline snapshot tests on deterministic stages.
- Citation validator has its own unit suite (out-of-range, renamed file, deleted file).
- LLM stages tested with recorded fixtures (no live calls in CI) + a nightly live eval workflow that reports the F5 metric.

## 11. Risks & open questions

- Token cost blowups on huge repos → hard budget per analysis with graceful "sampled" mode and visible coverage %.
- Framework heuristics are a long tail → registry pattern, ship top-8 frameworks, log unknowns for later.
- Mermaid degrades on dense graphs → cap + cluster, and offer dependency table as fallback.
