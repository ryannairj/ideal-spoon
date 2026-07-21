# Implementation Spec — Doc Drift

Read `PLAN.md` first. This file fixes the implementation decisions; do not re-decide them.

## 0. Fixed decisions

| Decision | Value |
|---|---|
| Name | **docdrift** (single Rust binary) · License MIT |
| Language | Rust stable, edition 2021; crates: `docdrift` (bin) + `docdrift-core` (lib) |
| Parsers | tree-sitter grammars vendored: markdown, typescript, tsx, javascript, python, go, rust, java, bash |
| Command check allowlist (word-0) | package.json scripts, Makefile/justfile targets, repo `bin/` + `scripts/` executables, and builtin allowlist `{npm,pnpm,yarn,npx,make,just,cargo,go,uv,pip,python,node,docker,git,gh,curl}` — allowlisted runners have their SUBcommand checked where knowable (`npm run X` → scripts[X]; `make X` → target X) |
| Command sources checked | fenced blocks tagged `bash|sh|shell|console|zsh` only; within them, lines starting `$ ` or first token of a plain line; comments/output lines ignored (console heuristic: lines without `$ ` in `console` blocks = output) |
| Symbol check | code-spans matching `[A-Za-z_][A-Za-z0-9_.]*\(\)` or `Class.method` patterns, resolved against tree-sitter symbol index; unresolved single-word spans are NOT errors (too noisy) — only spans that match an ex-symbol (existed in git history at HEAD~ or matches anchor) or anchored ones |
| Anchor comment syntax | `<!-- docdrift: key=value … -->` preceding the claim block; kinds: `symbol=path#Name [hash=…]`, `snapshot=path#RegionName hash=…` (region marked in code by `// docdrift-region: RegionName` … `// docdrift-endregion`), `file=path`, `cmd="…"`, `review-by=YYYY-MM-DD` |
| Snippet typecheck runners | `ts` → synthesized file + `tsc --noEmit` (repo tsconfig), `python` → `python -m py_compile`, `go` → `gofmt -e` parse check (not vet — no execution, fast); hidden setup lines via `<!-- docdrift-hidden: import x -->` |
| Severity config | `.docdrift.toml` per PLAN §7; exit non-zero iff any `error`-level finding |
| Semantic mode | any OpenAI-compatible endpoint (`DOCDRIFT_LLM_BASE/KEY/MODEL`); warn-only always; cache in `.docdrift-cache/` keyed `(prose_hash, code_hash)` |
| Outputs | human (default), `--format json|sarif`; PR-diff mode `--diff base..head` |
| Distribution | GitHub Action `docdrift/action@v1`, brew tap, cargo-binstall, pre-commit hook config |

## 1. Repository layout

```
docdrift/
  crates/docdrift-core/src/
    extract/{markdown.rs (claims from md AST), commands.rs, paths.rs, links.rs, symbols_spans.rs}
    anchors/{parse.rs, hash.rs, regions.rs}
    index/{symbols.rs (tree-sitter per lang), scripts.rs (pkg.json/make/just), files.rs}
    check/{commands.rs, paths.rs, links.rs, symbols.rs, snippets/{ts.rs, py.rs, go.rs}, freshness.rs}
    diffmode/{gitdiff.rs, renames.rs (symbol moved within file via ts), affected.rs}
    semantic/{client.rs, pairs.rs, cache.rs}
    config.rs report/{human.rs, json.rs, sarif.rs} suppress.rs
  crates/docdrift/src/main.rs        # clap: check, sync, annotate, stats
  action/{action.yml, entrypoint.sh}
  corpus/                            # FP corpus: git submodules of ~10 real repos + expected-zero manifests
  fixtures/{md/, anchors/, snippets/, semantic/{drifted/, insync/}}
```

## 2. Dependencies

`tree-sitter` + vendored grammar crates, `clap`, `serde`/`serde_json`/`toml`, `rayon` (parallel file scan), `similar` (diff hints), `git2`, `regex`, `sha2`, `reqwest` (semantic, feature-gated `semantic`), `walkdir`, `globset`. Dev: `insta` (snapshot tests), `assert_cmd`.

## 3. Claim model (core types)

```rust
enum Claim {
  Command { doc: Loc, line: String, word0: String, sub: Option<String> },
  Path    { doc: Loc, path: String },
  Link    { doc: Loc, target: String, anchor: Option<String> },
  Symbol  { doc: Loc, name: String, anchored: Option<AnchorRef> },
  Snippet { doc: Loc, lang: Lang, code: String, hidden_setup: Vec<String> },
  Anchored{ doc: Loc, anchor: Anchor },          // snapshot/file/cmd/review-by
}
struct Finding { claim: Claim, level: Level, code: &'static str /* e.g. CMD001 */,
  message: String, code_side: Option<Loc>, suggestion: Option<String> }
```
Finding codes (stable, documented in `docs/RULES.md`): CMD001 unknown command, CMD002 unknown script/target, PATH001 missing path, LINK001/002 broken link/anchor, SYM001 missing symbol, SYM002 signature-hash changed, SNAP001 snapshot mismatch, SNIP001 typecheck failed, FRESH001 review-by elapsed, SEM001 semantic drift (warn).

## 4. CLI contract

```
docdrift check [--config .docdrift.toml] [--diff base..head] [--format human|json|sarif]
               [--semantic] [--stats] [paths…]
docdrift sync [--all]        # interactive/bulk re-hash of intentional snapshot/symbol changes
docdrift annotate [--dry]    # suggests anchor insertions (code spans near headings w/ resolvable symbols)
```
Diff mode: (1) collect claims from changed docs (recheck all); (2) code-side changes: build changed-symbol set (git2 diff + tree-sitter old/new symbol tables incl. within-file moves and `git diff -M` renames) → re-check claims referencing them; (3) deleted/renamed symbols → grep docs corpus for mentions → SYM001 findings pointing at doc lines. Runtime budget: < 10 s on monorepo fixture PR (bench test).

## 5. Suppression & config

Inline: `<!-- docdrift-ignore: CMD001 reason="legacy doc kept for v1 users" -->` (reason mandatory — parser errors without it). Config `[ignore] patterns/claims` per PLAN. `--stats` prints findings-by-code + suppression count (FP telemetry for maintainers).

## 6. GitHub Action

Inputs: `mode: diff|full`, `semantic: bool`, `config`. Steps: install binary (release download, checksum), run check with SARIF output, upload SARIF (inline annotations on doc lines), optional sticky PR comment (marker `<!-- docdrift -->`) summarizing errors + semantic warnings; forks-without-secrets → semantic auto-off (no failure).

## 7. Milestone task lists

**M0** — T1 crate scaffold + config + md claim extraction (insta snapshots on `fixtures/md/` incl. nested fences, html blocks, tables); T2 index (scripts, files) + CMD/PATH/LINK checks + human report; T3 corpus submodules + expected-zero FP gate in CI (any FP = failure); T4 release workflow (musl static builds mac/linux/win). *Immediately useful.*
**M1** — T1 tree-sitter symbol index (8 langs) + SYM001; T2 anchors parse + symbol sigHash (normalized signature text) + SYM002; T3 snapshot regions + SNAP001 + `sync` (interactive: show old/new, confirm re-hash; `--all` bulk); T4 freshness FRESH001; T5 anchor round-trip tests (break → fail; sync → pass).
**M2** — T1 gitdiff + changed-symbol sets + rename/move detection; T2 affected-claim mapping + deleted-symbol doc grep; T3 SARIF + JSON emitters (SARIF validated against schema); T4 action/ + sticky comment + fixture-repo workflow E2E in this repo's CI; T5 monorepo PR bench < 10 s.
**M3** — T1 snippet synthesis (import resolution against repo, hidden-setup) + ts/py/go runners (never-executes enforced: runners construct argv w/o shell, code never invoked — design test); T2 SNIP001 + renamed-export fixture test; T3 `annotate` heuristics + `--dry` preview.
**M4** — T1 semantic mode: pair prose paragraphs adjacent to anchors with anchored code region → drift prompt → SEM001 warnings w/ dual citations; cache; T2 semantic fixture eval (15 drifted ≥ 12 caught, 15 in-sync ≤ 2 false — recorded in CI, live nightly); T3 `--stats`, suppression-reason enforcement, docs/RULES.md; T4 brew/binstall/pre-commit packaging. Tag v1.0.

## 8. Test mapping

The corpus FP gate is the covenant (PLAN §10) — runs on every PR. insta snapshots per extractor/checker. `assert_cmd` CLI-level tests for exit codes/formats. Semantic eval nightly.

## 9. Algorithm details

**Symbol-matching heuristic (`check/symbols.rs`) — exact, not fuzzy.** Doc code-spans are matched against the tree-sitter symbol index by *exact* identifier, never edit-distance (fuzzy matching produces false drift). The dotted-path is resolved left-to-right:

```
function resolveSpan(span):                 # span already matched CMD/symbol regex in §0
  name = strip trailing "()" from span
  parts = name.split(".")
  if parts.len == 1:                         # bare "foo"
    hits = index.byName(parts[0])            # exact identifier match across langs
    if hits.empty: return NotAClaim          # unresolved single word → ignored (never SYM001)
    return Resolved(hits)
  # dotted: "Class.method" | "module.func" | "pkg.Type.Method"
  container = index.byName(parts[0])
  if container.empty: return NotAClaim       # unknown container → not our claim (avoid noise)
  member = index.member(container, parts[1..])   # walk members via ts child scopes
  return member.empty ? Missing(name) : Resolved(member)
```

A span becomes a **SYM001** only when (a) it is anchored, or (b) `parts[0]` resolves to a known container/symbol but the member is gone, or (c) the name existed at `HEAD~`/anchor and no longer exists. Unanchored bare words that never resolved are silently ignored — this is the noise-control covenant.

**Snippet synthesis & import resolution (`check/snippets/*.rs`).** To typecheck a fenced snippet without executing it:

```
function synthesize(snippet, lang):
  body = hidden_setup_lines(snippet) ++ snippet.code   # <!-- docdrift-hidden: ... -->
  imports = collect import/use/require statements from body
  for imp in imports:
    if resolvable against repo (tsconfig paths / PYTHONPATH / go.mod module root):
      keep as-is (real import → real type check)
    else:
      leave unresolved → the runner reports it as SNIP001 (broken doc dependency)
  write body to a temp file under the repo root (so relative imports & tsconfig resolve)
  run: ts → `tsc --noEmit` | py → `python -m py_compile` | go → `gofmt -e`
  argv built without a shell; snippet code is NEVER invoked/run (design test enforces)
```

Import resolution therefore uses the *repo's own* config (tsconfig `paths`, go module root, package layout); unresolved imports are a legitimate SNIP001 rather than a crash.

**Semantic cache key granularity.** The cache key is `sha256(normalize(prose_paragraph) || "\x00" || normalize(code_region))` where `normalize` = trim + collapse internal whitespace + strip trailing comments; the unit of caching is **one prose-paragraph × anchored-code-region pair** (not the whole file). Changing either side invalidates only that pair's entry. Model id is folded into the key (`model || key`) so switching `DOCDRIFT_MODEL` doesn't serve stale verdicts. Entries stored as JSON files in `.docdrift-cache/<first2>/<hash>.json`.
