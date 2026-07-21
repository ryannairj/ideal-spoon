# Implementation Spec — Depwatch

Read `PLAN.md` first. This file fixes the implementation decisions; do not re-decide them.

## 0. Fixed decisions

| Decision | Value |
|---|---|
| Name | **Depwatch** (`depwatch` CLI, GitHub App `depwatch-bot`) · License Apache-2.0 |
| Stack | TS 5 + Node 22, pnpm workspaces: `packages/core` (all logic), `apps/cli`, `apps/server` (Fastify+Probot), `apps/dashboard` deferred (no dashboard in MVP — server is headless) |
| CLI cache | SQLite `~/.cache/depwatch/cache.db` (better-sqlite3); server: Postgres 16 (Drizzle) |
| Models | distill: `deepseek-chat`; affected-callsite reasoning: `claude-sonnet-5`; registry per CONVENTIONS |
| Evidence order | GitHub Releases → `CHANGELOG*.md` at tags → commit messages+PR titles (fallback, confidence downgrade) |
| API diff | `.d.ts` exported-surface extraction via `ts-morph`; surface = exported symbols + signature hash (normalized text of type) |
| Usage scan | tree-sitter (ts/js incl. tsx); import styles per PLAN F3; confidence: `static > require > reexport > dynamic(unknown)` |
| Verdicts | `breaking-for-you` \| `review` \| `safe`; any `unknown` usage confidence caps verdict at `review` (never `safe`) |
| Exit codes | 0 safe, 1 review, 2 breaking-for-you, 3 error |
| Changes schema (zod) | `{kind:'breaking'|'feature'|'fix'|'deprecation', summary, symbols:string[], evidence_url, confidence}` — `breaking` requires evidence_url; api-diff removals injected as breaking with `evidence_url:'api-diff'` |
| PR comment | single sticky comment marker `<!-- depwatch -->`, edited in place |
| Batch rules (audit) | batch 1 = all patch bumps; batch 2 = minors grouped by ecosystem area; each major = solo, ordered by exposure score `risk×usage_sites×staleness` |
| Config | `.depwatch.yml` optional: `ignore: [pkgs]`, `budget_tokens`, `comment: on|off`, `status_check: on|off` |

## 1. Repository layout

```
depwatch/
  packages/core/src/
    evidence/{registry.ts (npm meta), releases.ts, changelog.ts, commits.ts, bundle.ts}
    apidiff/{extract.ts (ts-morph surface), diff.ts, hash.ts}
    distill/{prompt.ts, validate.ts}
    usage/{scan.ts (tree-sitter), imports.ts, sites.ts}
    verdict/{join.ts, render/{terminal.ts, markdown.ts, json.ts}}
    lockfile/{npm.ts, pnpm.ts, yarnBerry.ts, detect.ts}
    cache.ts  llm/  types.ts (zod)
  apps/cli/src/{index.ts (commander), check.ts, audit.ts}
  apps/server/src/{app.ts, webhook.ts, prDetect.ts, comment.ts, statusCheck.ts, store.ts}
  bench/{bumps.yaml (20 labeled), repos/ (fixture tarballs: zod3, express4, eslint8 users), run.ts}
  fixtures/{lockfiles/, changelogs/, dts/}
```

## 2. Dependencies

`commander, ts-morph, web-tree-sitter, zod, better-sqlite3, drizzle-orm, postgres, probot, fastify, @octokit/rest, semver, yaml, tiktoken, pacote` (registry fetch + tarball extraction for .d.ts), `chalk, cli-table3`. Dev: vitest, playwright not needed (no UI).

## 3. Configuration

CLI env: `DEEPSEEK_API_KEY, ANTHROPIC_API_KEY, GITHUB_TOKEN` (evidence fetch rate limits), `DEPWATCH_CACHE_DIR`. Server adds Probot vars + `DATABASE_URL`.

## 4. Cache/DB schema

```sql
-- shared shape (SQLite CLI / Postgres server)
CREATE TABLE evidence_bundles (pkg text, from_v text, to_v text, sources jsonb,
  changes jsonb, api_diff jsonb, confidence text, model text, distilled_at timestamptz,
  PRIMARY KEY(pkg, from_v, to_v));
CREATE TABLE checks (id uuid PK, repo text, pkg text, from_v text, to_v text,
  usage jsonb, affected jsonb, verdict text, created_at timestamptz);
-- server only
CREATE TABLE installations (id bigint PK, settings jsonb);
CREATE TABLE pr_comments (repo text, pr int, comment_id bigint, head_sha text, PRIMARY KEY(repo, pr));
```

## 5. Pipeline contracts

**evidence.bundle(pkg, from, to)**: registry versions in `(from, to]` → per source in §0 order collect raw texts scoped to that range (release-tag matching by semver; changelog section slicing by version headers) → distill per version-chunk (≤ 8k tokens/chunk) → merge + zod validate (invalid breaking-without-evidence → downgrade to `review`-class `feature` with note) → inject api-diff breakings → cache.

**apidiff.extract(pkg@v)**: `pacote extract` → locate types entry (`types`/`exports.types`/bundled `.d.ts`; `@types/<pkg>` fallback) → ts-morph project → exported surface map `{name → {kind, sigHash}}`. `diff`: removed (breaking), changed sigHash (breaking-candidate), added (info). No types → `{unavailable: true}`. sigHash normalization and overload merging are pseudocoded in §9.

**distill(pkg, from, to, rawTexts)**: chunk raw evidence ≤ 8k tokens → call LLM with the committed template `fixtures/prompts/distill.md` (see §9) → parse JSON → zod-validate against the Changes schema (§0). The citation rule (`breaking` requires `evidence_url`) is enforced post-parse, not left to the model.

**usage.scan(repoDir, pkg)**: workspace-aware (respect `pnpm-workspace.yaml`/`workspaces` globs; attribute per package); find import sites → member usage: named imports tracked to identifiers used; namespace imports → `ns.member` property accesses; default import → all-usages-of-binding; dynamic import w/o static members → `unknown`. Output sites `[{file, line, snippet, symbols[], confidence}]`.

**verdict.join(changes, usage)**: affected = changes whose `symbols ∩ usedSymbols ≠ ∅` (symbol match: exact, plus dotted-suffix match `Foo.bar`~`bar` flagged lower confidence); breaking∩used → breaking-for-you; deprecation/behavioral∩used → review; else safe (subject to unknown-cap rule).

## 6. CLI & server surfaces

`depwatch check <pkg>[@range|latest] [--json] [--md] [--no-llm (api-diff+changelog-raw only)]` · `depwatch audit [--budget-tokens N] [--json]` → ranked table + batches · `depwatch ci` (alias check for all lockfile changes vs base branch, used in Actions). Server: webhook `pull_request` → `prDetect.ts` (branch prefixes `renovate/`, `dependabot/`, actor bots + lockfile diff parse) → per changed dep run pipeline (shared bundle cache) → render markdown → sticky comment upsert + optional non-blocking status `depwatch/verdict`.

## 7. Milestone task lists

**M0** — T1 workspaces scaffold + CLI skeleton + cache; T2 registry + evidence collectors (fixture changelogs/releases; version-range slicing tests); T3 distill + zod + citation rule; T4 terminal renderer. *Ships changelog summarizer (`--no-usage` implicit).* 
**M1** — T1 pacote extract + ts-morph surface (unit suite: overloads → merged sigHash, namespaces, type-only exports, `export * from`); T2 diff + breaking injection + types-missing degradation path; T3 confidence labeling through renderer.
**M2** — T1 tree-sitter usage scan + import-style matrix tests; T2 workspace attribution (pnpm fixture); T3 verdict join + unknown-cap rule; T4 golden fixtures: zod3→4, express4→5, eslint8→9 user-repos → exact affected-site lists; T5 `--json` + exit codes + timing budget test (cached < 5 s). *Real product ships.*
**M3** — T1 Probot server + prDetect (fixtures per bot) + lockfile diff parsers (npm/pnpm/yarn-berry fuzz fixtures); T2 markdown renderer + sticky comment + rebase re-run; T3 status check + config file; T4 shared Postgres bundle cache.
**M4** — T1 audit mode (outdated scan via registry, ranking formula unit-tested, batches, budget guard); T2 `depwatch ci` + GitHub Action yaml + docs; T3 bench harness as CI gate (recorded: breaking-recall ≥ 0.9, safe-FP = 0 on `bench/bumps.yaml`; nightly live); T4 self-host compose + README. Tag v1.0. *(M5: PyPI adapter — interface `EcosystemAdapter` already extracted in core.)*

## 8. Test mapping

`bench/` = asymmetric quality gate (wrong-safe = build fail). Surface-extraction and lockfile parsers get fuzz/fixture suites. Recorded-LLM fixtures throughout; nightly live with cost report.

Benchmark expected outputs are committed, not just referenced: each entry in `bench/bumps.yaml` links to a fixture directory `bench/repos/<case>/` plus an `expected.json` (the exact verdict + affected-site list). The harness (`bench/run.ts`) diffs actual vs `expected.json` and fails on any regression. Example `bumps.yaml` entry:

```yaml
- case: zod3-to-4
  pkg: zod
  from: 3.23.8
  to: 4.0.0
  repo: repos/zod3-users/      # fixture tarball dir
  expected: repos/zod3-users/expected.json
```

## 9. Algorithms (pseudocode) & prompt fixtures

**sigHash normalization + overload merging** — a symbol with N call/construct signatures (overloads) collapses to one stable hash so that reordering overloads is not a false "changed":

```
function sigHash(symbol):
  sigs = []
  for each overload in symbol.callSignatures ∪ symbol.constructSignatures:
    t = ts-morph type text of overload           # e.g. "(a: string, b?: number): Foo"
    t = stripComments(t)
    t = collapseWhitespace(t)                     # runs of space/newline → single space
    t = normalizeParamNames(t)                    # a,b,... → positional $0,$1 (names are not API)
    t = expandOneLevelAlias(t)                    # resolve type aliases one level; stop (no deep recursion)
    sigs.push(t)
  sigs = sort(sigs)                               # order-independent: overload reordering is not a change
  body = symbol.kind + "|" + sigs.join("§")
  return sha256(body).slice(0, 16)
```

Removal of any overload changes the joined body → changed sigHash → breaking-candidate. Adding an overload also changes the hash but is classified `added`/info by comparing signature-set membership (removed-overload set non-empty ⇒ breaking).

**lockfile diff parsing** — one function per manager (`npm.ts`/`pnpm.ts`/`yarnBerry.ts`), all returning `{pkg, from, to}[]`:

```
function diffLockfile(baseText, headText, manager):
  base = manager.parse(baseText)   # → Map<pkgKey, version>; pkgKey = name (+ path for workspaces)
  head = manager.parse(headText)
  changes = []
  for key in head.keys():
    if key not in base: continue           # newly added dep → not a "bump", skip (no from-version)
    if base[key] != head[key]:
      changes.push({ pkg: nameOf(key), from: base[key], to: head[key] })
  # removals (key in base, not head) are ignored: nothing to review for a removed dep
  return dedupeByPkgKeepingWidestRange(changes)
```

Manager-specific `parse`: npm reads `packages` map keyed by `node_modules/...` paths (v2/v3 lockfileVersion); pnpm reads `packages:` keys `/name@version`; yarn-berry reads YAML resolution entries. Malformed or unknown `lockfileVersion` → `parse` throws `LockParseError` (handled in §10).

**Prompt fixture** — `fixtures/prompts/distill.md` (v1) pins: system role ("changelog distiller, cite or omit"), input variables `{{pkg}} {{from}} {{to}} {{rawChunk}}`, required output = JSON array of the Changes schema, and one worked example showing a `breaking` entry with a real `evidence_url` and a `feature` entry without one.

## 10. Error Recovery & Graceful Degradation

| Failure | Trigger | Backoff | Fallback | User-facing UX |
|---|---|---|---|---|
| Registry/evidence fetch (npm, GitHub) | HTTP 5xx / network error / timeout 10s | 3 retries, base 500ms, ×2, cap 5s, full jitter; 429 honors `Retry-After` | Drop that evidence source, continue with remaining sources at downgraded confidence; if all sources fail, verdict caps at `review` | CLI: yellow "evidence degraded" note in report; server sticky comment adds "⚠ partial evidence" line |
| LLM distill | timeout 60s / non-JSON / zod-invalid | 1 retry with the parse error appended to the prompt | Fall back to api-diff + raw-changelog-only mode (as `--no-llm`); mark chunk `distill_failed` | Report footer: "summaries unavailable for N versions (raw links shown)" |
| `.d.ts` extraction | `pacote` extract fails / no types entry | no retry | `{unavailable:true}` → api-diff contributes nothing; rely on changelog evidence only | Report notes "API surface unavailable" |
| Lockfile parse | `LockParseError` / unknown lockfileVersion | no retry | Skip that lockfile, process others; if the PR's only lockfile is unparseable, post neutral comment | Comment: "could not parse <file> (lockfileVersion X)" |
| Token budget hit (audit) | cumulative tokens ≥ `budget_tokens` | n/a | Stop distilling, emit partial ranked table with remaining deps marked `not-analyzed`, resumable next run | "budget reached: N of M analyzed" |
