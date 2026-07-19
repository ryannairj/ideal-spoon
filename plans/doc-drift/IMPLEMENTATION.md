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
