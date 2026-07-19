# Open Canvas — Claude-style artifacts/design studio on any model

**Category:** Clone / AI tool · **Difficulty:** Medium–Hard · **Platform:** Web app, self-hostable

## 1. Vision

The magic of Claude's artifacts/design mode — chat on the left, live rendered creation on the right, iterated conversationally — but **model-agnostic and self-hosted**. Plug in DeepSeek, GLM, Kimi, Qwen, local Ollama, or any OpenAI-compatible endpoint; get the same "ask → see → refine" loop for HTML pages, React components, SVG, diagrams, documents, and small games. The cheap-model era makes this compelling: 90% of the artifact experience at 2% of the token price.

## 2. Why it can win

- Vendor UIs lock the loop to their model. Open Canvas decouples the **experience** (preview sandbox, versioning, targeted edits) from the **model**.
- The differentiating engineering is the *edit discipline*: cheap models rewrite whole files badly. We enforce a structured edit protocol + auto-repair loop that makes weaker models look far better than raw chat.
- Model bake-off built in: same prompt across 3 models side-by-side. Nobody offers this for artifacts.

## 3. Users & use cases

Tinkerers, developers prototyping UI, educators, cost-sensitive teams.

1. "Make a landing page for my sourdough bakery" → rendered page in seconds → "warmer palette, add a menu section" → in-place update.
2. Build a small React dashboard component against pasted JSON.
3. Compare DeepSeek vs GLM vs Kimi output for one brief; fork the winner.
4. Export artifact as standalone HTML / component file / zip.

## 4. MVP scope & non-goals

**MVP:** chat + live preview; artifact types: single-file HTML/CSS/JS, React component (esbuild-wasm in-browser bundling), SVG, Markdown doc, Mermaid; provider manager (OpenAI-compatible + Anthropic + Ollama); versions & diffs; targeted-edit protocol with repair loop; export; local accounts.
**Non-goals:** multi-file projects/repos (that's an IDE agent's job), backend code execution (client-side sandbox only), collaboration/multiplayer, image generation, mobile.

## 5. Tech stack

- **App:** Next.js 15 + TypeScript; Postgres (Drizzle) or SQLite in self-host mode; streaming via SSE.
- **Sandbox:** artifacts render in a sandboxed `<iframe>` (`sandbox="allow-scripts"`, strict CSP, no network by default; opt-in allowlist for CDN imports via import-map). React artifacts bundled in-browser with esbuild-wasm.
- **LLM layer:** provider registry (base URL + key + model + capabilities flags: context length, supports-prefill, tokens/s estimate); one streaming interface over all.
- **Diffing:** unified-diff apply engine with fuzzy matching (context-anchored), plus AST-assisted fallback for JS/TSX via Babel when hunks fail.

## 6. Architecture & the edit protocol (the core IP)

```
chat turn → router: NEW artifact | EDIT artifact | ANSWER only
NEW  → generate full file (streaming into preview)
EDIT → model must emit SEARCH/REPLACE blocks (Aider-style) against current version
     → apply engine (exact → whitespace-fuzzy → anchor-fuzzy → AST fallback)
     → on apply failure: auto-repair prompt with failed hunk + real context (≤ 2 retries)
     → on runtime error in preview (window.onerror piped out): auto-fix prompt with error + stack (≤ 2 retries, user-visible "fixing…" state)
every applied change = new immutable version (with the prompt that caused it)
```

Renderer contract per artifact type (what wrapper HTML, what libraries are pre-allowlisted — e.g. Tailwind via CDN, React 19 UMD, Mermaid) is versioned so old artifacts keep rendering.

## 7. Data model

- `providers(id, user_id, name, kind{openai_compat,anthropic,ollama}, base_url, api_key_enc, model, caps_json, price_json)`
- `projects(id, user_id, title, artifact_type, active_version_id, created_at)`
- `versions(id, project_id, parent_id, content, created_by{model,user}, prompt, model_used, tokens_in, tokens_out, created_at)`
- `messages(id, project_id, role, content, version_id NULLABLE, at)`
- `bakeoffs(id, prompt, entries_json /*[{provider_id, version_id, votes}]*/)`

## 8. Feature specs

**F1 — Chat + live preview.** Split pane; generation streams into a code view, preview refreshes on completion (and progressively for HTML); artifact type auto-detected from request with manual override. *AC:* time-to-first-render < 1.5× model generation time; preview iframe cannot reach the parent origin or network (CSP test suite).

**F2 — Targeted edits.** Edit protocol as §6. *AC:* benchmark of 40 recorded edit tasks across 3 cheap models: ≥ 90% end in a successfully applied change (incl. repair loop); zero silent corruptions (every apply is validated by re-parse for JS/TSX/SVG/JSON).

**F3 — Versions & diffs.** Timeline rail; click = preview that version; diff view between any two; restore/fork; "which prompt caused this" always visible. *AC:* restore never mutates history (immutable chain verified by tests).

**F4 — Provider manager & bake-off.** CRUD providers, connection test, per-project model picker, cost ticker per message (from price_json); bake-off mode: 1 prompt → N providers → grid of previews → pick winner to continue. *AC:* Ollama local endpoint works with zero cloud calls; cost ticker within 5% of provider-reported usage.

**F5 — Manual edit + user/AI interleave.** Monaco editor on the code pane; user edits create user-versions; next AI edit works from the latest content. *AC:* AI edit after manual edit doesn't clobber user changes outside its hunks (test with fixture).

**F6 — Export & share.** Download HTML/tsx/svg/md; zip with README; read-only share link rendering the sandboxed artifact. *AC:* exported HTML is truly standalone (opens from `file://`, no network unless artifact used allowlisted CDN — then documented in export README).

## 9. Milestones

- **M0 (1):** scaffold, provider layer with streaming, chat UI, HTML artifact full-generation + sandboxed preview. *Ships: core demo.*
- **M1 (2):** edit protocol + apply engine + repair loops; versions/diffs. *Ships: the iteration loop.*
- **M2 (3):** React/SVG/Markdown/Mermaid renderers, Monaco manual edits, runtime-error auto-fix. 
- **M3 (4):** provider manager UX, cost tracking, bake-off mode. *Ships: MVP.*
- **M4 (5):** export/share, edit-benchmark harness as CI gate, self-host docker-compose + docs. *Ships: v1.0.*

## 10. Testing

- Apply-engine unit suite: exact/fuzzy/AST paths, malformed blocks, overlapping hunks, unicode.
- Recorded-model fixtures for CI; nightly live benchmark (F2 metric) across DeepSeek/GLM/Kimi with cost report.
- Sandbox security tests: escape attempts (parent access, fetch, form nav, top-nav) must fail — release-blocking.
- Playwright: generate → edit → restore → export happy path.

## 11. Risks & open questions

- Weak models ignore the edit format → per-provider capability flag can force "full regenerate" mode for models that fail the benchmark; still versioned so UX holds.
- CDN-dependent artifacts break offline → default no-network, explicit allowlist per artifact, export README lists deps.
- Prompt-injection via pasted content into artifacts is contained by sandbox (no cookies/origin/network) — document the model.
