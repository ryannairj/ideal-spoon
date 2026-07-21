# Implementation Spec — Open Canvas

Read `PLAN.md` first. This file fixes the implementation decisions; do not re-decide them.

## 0. Fixed decisions

| Decision | Value |
|---|---|
| Name | **Open Canvas** (`opencanvas`) · License Apache-2.0 |
| Stack | Next.js 15 + TS 5; DB: Drizzle with SQLite (`better-sqlite3`) default, Postgres via `DATABASE_URL` switch; no Redis (in-process streaming) |
| Auth | local email+password (argon2id) — self-host-first; single-user mode flag `SINGLE_USER=true` skips auth |
| Edit protocol | Aider-style SEARCH/REPLACE blocks (format fixed below); apply ladder: exact → whitespace-insensitive → context-anchor fuzzy (first/last 2 lines) → Babel AST patch (js/tsx only) |
| Sandbox | `<iframe sandbox="allow-scripts" csp="default-src 'none'; script-src 'unsafe-inline' blob:; style-src 'unsafe-inline'; img-src data: blob:">` served from separate origin path `/sandbox/[versionId]` with strict CSP headers; CDN allowlist via import-map injection when artifact opts in |
| Artifact types + renderer contract v1 | `html` (raw), `react` (esbuild-wasm bundle, React 19 UMD inlined), `svg`, `markdown` (marked + own CSS), `mermaid` (inlined lib) — wrapper templates versioned in `renderers/v1/` |
| Editor | Monaco (`@monaco-editor/react`) |
| Providers | kinds: `anthropic`, `openai_compat`, `ollama`; keys AES-GCM encrypted with `APP_SECRET` |
| Streaming | SSE from `/api/generate` |
| Ports | 3000 |

## 1. Repository layout

```
opencanvas/
  src/
    app/
      page.tsx                     # project list + new
      p/[projectId]/page.tsx       # main split view
      bakeoff/page.tsx
      settings/providers/page.tsx
      sandbox/[versionId]/route.ts # serves wrapped artifact w/ CSP headers
      api/{generate/route.ts (SSE), apply/route.ts, providers/route.ts,
           projects/…, versions/…, export/[versionId]/route.ts, share/[token]/route.ts}
    core/
      router.ts                    # NEW | EDIT | ANSWER classification (LLM w/ fallback heuristic)
      editblocks/{parse.ts, apply.ts, fuzzy.ts, astApply.ts, validate.ts}
      repair.ts                    # failed-hunk + runtime-error repair loops (max 2 each)
      renderers/v1/{html.ts, react.ts (esbuild-wasm), svg.ts, markdown.ts, mermaid.ts, wrap.ts}
      llm/{registry.ts, anthropic.ts, openaiCompat.ts, ollama.ts, stream.ts, cost.ts}
      detect.ts                    # artifact-type detection from request
    components/{ChatPane, CodePane (Monaco), PreviewFrame, VersionRail, DiffView,
                ProviderForm, ModelPicker, CostTicker, BakeoffGrid, TypeBadge, ErrorFixBanner}
  bench/edits/{tasks/*.yaml, run.ts}   # 40 edit tasks, per-model apply-success metric
  test/sandbox-escape/*.spec.ts
```

## 2. Dependencies

`next, react, drizzle-orm, better-sqlite3, postgres, zod, @monaco-editor/react, esbuild-wasm, marked, mermaid (vendored build), diff (jsdiff), @babel/parser+traverse+generator, nanoid, tailwindcss, argon2`. Dev: vitest, playwright.

## 3. Edit protocol (normative)

Model must output edits as:
```
<<<<<<< SEARCH
(exact current content)
=======
(replacement)
>>>>>>> REPLACE
```
Rules enforced by `parse.ts`: 1+ blocks; SEARCH nonempty (except append form `SEARCH` empty + marker `@@append`); blocks applied in order; overlapping matches rejected. `apply.ts` ladder per §0; every successful apply → re-parse validation (babel for js/tsx, XML parse for svg, JSON.parse for json, no-op for md/html) — parse failure = apply failure → repair loop with real error. Providers failing bench below the `@TUNE(fullRewriteThreshold=0.60)` apply-success rate get `capabilities.fullRewriteOnly=true` → edit turns regenerate whole file (still versioned). The threshold is calibrated in M4 T4 against `bench/edits` recorded results; until a provider has a bench score it is treated as capable (blocks first, repair on failure).

## 4. Database schema

```sql
CREATE TABLE users (id text PK, email text UNIQUE, pw_hash text, created_at int);
CREATE TABLE providers (id text PK, user_id text, name text, kind text CHECK(kind IN ('anthropic','openai_compat','ollama')),
  base_url text, api_key_enc blob, model text, caps_json text DEFAULT '{}', price_json text DEFAULT '{}');
CREATE TABLE projects (id text PK, user_id text, title text, artifact_type text,
  active_version_id text, provider_id text, created_at int);
CREATE TABLE versions (id text PK, project_id text NOT NULL, parent_id text,
  content text NOT NULL, created_by text CHECK(created_by IN ('model','user')),
  prompt text, model_used text, tokens_in int, tokens_out int, renderer_version text DEFAULT 'v1',
  created_at int);
CREATE TABLE messages (id text PK, project_id text, role text, content text,
  version_id text, at int);
CREATE TABLE shares (token text PK, version_id text, created_at int);
CREATE TABLE bakeoffs (id text PK, user_id text, prompt text, entries_json text, created_at int);
```

## 5. API contract

| Route | Behavior |
|---|---|
| `POST /api/generate` `{projectId, message, providerId?}` | SSE: `{t:'route', mode}` → `{t:'delta', text}` (code stream) → `{t:'applied', versionId}` \| `{t:'apply_failed', hunks}` → repair deltas → `{t:'done', usage, cost}` \| `{t:'answer', text}` for ANSWER mode |
| `POST /api/apply` `{projectId, content}` | manual (Monaco) save → user version |
| `POST /api/versions/{id}/restore` | new version copying content (immutability test) |
| `GET /api/versions/{id}/diff/{otherId}` | unified diff |
| `GET /api/export/{versionId}?format=file|zip` | standalone artifact (README lists any CDN deps) |
| `POST /api/providers/test` | round-trip "say ok" call, returns latency |
| `POST /api/bakeoff` `{prompt, providerIds[]}` | N parallel generations → entries with versionIds |
| runtime errors | PreviewFrame postMessage `{type:'runtime_error', message, stack}` → client POSTs to generate with `mode:'fix_error'` (banner shows "fixing…", max 2) |

## 6. UI

Main split: ChatPane left (messages, cost ticker per turn), right tabs [Preview | Code | Diff] + VersionRail bottom (dots per version, hover = prompt tooltip, click = view, fork/restore buttons). TypeBadge with manual override dropdown (re-detect warning). Bakeoff: prompt box, provider multi-select, grid of PreviewFrames + pick-winner → creates project from winner. Providers settings: table + form + test button + price editor.

## 7. Milestone task lists

**M0** — T1 scaffold + schema + auth/single-user; T2 provider registry + streaming (3 kinds) + ProviderForm/test; T3 router (heuristic first: "make/create/build" + no active artifact → NEW) + html full generation streaming into CodePane; T4 sandbox route + CSP + PreviewFrame; T5 CI. *Core demo ships.*
**M1** — T1 editblocks parse/apply ladder + unit suite (exact/fuzzy/overlap/unicode/malformed); T2 repair loop (apply-fail path); T3 versions (immutable chain, restore/fork) + VersionRail + DiffView; T4 zod-validated re-parse validation per type.
**M2** — T1 react renderer (esbuild-wasm in web worker, UMD inline) + svg/markdown/mermaid renderers + renderer_version stamping; T2 Monaco manual edits → user versions + interleave test (AI edit after user edit preserves outside-hunk content); T3 runtime-error auto-fix loop + ErrorFixBanner; T4 sandbox-escape test suite (parent access, fetch, form nav, top nav, window.open — all must fail).
**M3** — T1 model picker per project + CostTicker (usage × price_json, 5% reconciliation test vs provider-reported); T2 bakeoff mode; T3 ollama zero-cloud path verified test.
**M4** — T1 export (standalone html/tsx/svg/md; zip w/ README; `file://` open test); T2 share tokens (read-only sandbox page); T3 `bench/edits` harness (40 tasks × 3 cheap models; CI recorded-fixture ≥90% apply-success gate; nightly live w/ cost report); T4 capability auto-flag from bench results; T5 docker-compose + docs. Tag v1.0.

## 8. Test mapping

Release-blocking: sandbox-escape suite, apply-validation (zero silent corruption — every apply re-parses), version immutability. Bench harness is the F2 gate. Playwright: generate → edit → restore → export happy path.

## 9. Intent routing, repair loops & prompt fixtures

**Intent classification (`router.ts`)** — heuristic-first with an LLM tiebreak and a defined fallback so the failure mode is never undefined:

```
function classify(message, hasActiveArtifact):
  m = message.toLowerCase()
  # 1. Deterministic heuristics (fast path, no LLM)
  if not hasActiveArtifact and matches(m, /\b(make|create|build|generate|new)\b/):
    return NEW
  if hasActiveArtifact and matches(m, /\b(change|edit|fix|update|add|remove|rename|refactor)\b/):
    return EDIT
  if matches(m, /^\s*(what|why|how|explain|does|can|is|are)\b/) or endsWith(m, "?"):
    return ANSWER
  # 2. Ambiguous → single cheap LLM call returning one token: NEW|EDIT|ANSWER
  label = llmClassify(message, hasActiveArtifact)   # fixtures/prompts/route.md
  if label in {NEW, EDIT, ANSWER}: return label
  # 3. Fallback when LLM errors or returns garbage (never throw):
  return hasActiveArtifact ? EDIT : NEW
```

**Repair loops (`repair.ts`, max 2 each)** — "real context" passed to the repair prompt is precisely: (a) the failed SEARCH block(s) verbatim, (b) the surrounding ±8 lines of current file content around the best fuzzy-match anchor (or whole file if <120 lines), and (c) the concrete failure reason ("SEARCH not found" | parse error message | runtime error+stack). It is never the whole conversation.

**Prompt fixtures** (committed under `fixtures/prompts/`, all versioned `# v1`):
- `generate.md` — NEW-artifact generation; vars `{{artifactType}} {{userMessage}}`; output = raw artifact body.
- `edit.md` — EDIT turn; vars `{{artifactType}} {{currentContent}} {{userMessage}}`; output = SEARCH/REPLACE blocks per §3.
- `edit-repair.md` — apply-failure repair; vars `{{failedBlocks}} {{contextWindow}} {{reason}}`; output = corrected SEARCH/REPLACE blocks only.
- `runtime-fix.md` — runtime-error repair; vars `{{currentContent}} {{errorMessage}} {{stack}}`; output = SEARCH/REPLACE blocks.
- `route.md` — classification tiebreak; vars `{{message}} {{hasActiveArtifact}}`; output = exactly one of `NEW|EDIT|ANSWER`.

**Renderer contract versioning** — each version row stamps `renderer_version` (default `v1`). Wrappers live in `renderers/v1/`. A breaking wrapper change ships as `renderers/v2/` with `v1` retained verbatim; old versions keep rendering under their stamped version. There is no in-place migration — stamps are immutable, so historical artifacts never re-render differently.

## 10. Error Recovery & Graceful Degradation

| Failure | Trigger | Backoff | Fallback | User-facing UX |
|---|---|---|---|---|
| Provider stream error | non-2xx / socket drop / timeout 120s mid-stream | 1 retry of the turn if 0 deltas received; no retry once deltas started (would duplicate) | emit `{t:'error', message}`; keep last good version unchanged | ChatPane inline error bubble + "Retry" button |
| Edit apply fails | SEARCH not found / re-parse fails | repair loop, max 2, each with real context (§9) | after 2 fails: discard the turn, keep prior version; if provider is `fullRewriteOnly`, do a full regenerate instead | `{t:'apply_failed'}` → banner "couldn't apply edit, kept previous version" |
| Runtime error in preview | PreviewFrame postMessage `runtime_error` | auto-fix loop, max 2 | after 2 fails: stop, show error, leave artifact as-is | ErrorFixBanner: "fixing…" → "couldn't auto-fix" with error text |
| Router LLM error | classify tiebreak call fails | no retry | deterministic fallback (`§9`: EDIT if artifact exists else NEW) | silent; user sees the resulting action |
| Provider unreachable at test | `providers/test` non-2xx/timeout 15s | no retry | mark provider `unverified`; block generation with clear message | Providers table shows red "failed" with error |
| esbuild-wasm bundle failure | react artifact fails to compile | no retry | render compile error into PreviewFrame body (not blank) | Preview shows the esbuild error text |
