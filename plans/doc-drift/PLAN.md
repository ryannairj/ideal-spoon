# Doc Drift — CI guard that catches documentation lying about the code

**Category:** Dev tool · **Difficulty:** Medium · **Platform:** CLI + GitHub Action + PR bot

## 1. Vision

Docs rot silently: the README's install steps reference a deleted script, the API guide shows a parameter renamed last quarter, the architecture doc describes a service that no longer exists. Doc Drift links doc claims to code anchors and screams in CI when they diverge — "README.md:42 says `npm run migrate` but no such script exists" — turning documentation freshness from a virtue into a check.

## 2. Why it can win

- Everyone has this problem; the only current fix is discipline (lol). Existing tools check links and spelling, not *truth*.
- Two-layer design keeps it trustworthy: **deterministic checkers** (commands, paths, flags, code snippets, API signatures — zero false-positive tolerance) fail CI; **LLM semantic drift detection** (prose descriptions vs implementation) only *warns* on the PR. Determinism where we gate, AI where we hint.

## 3. Users & use cases

OSS maintainers, platform teams owning runbooks, anyone with a `docs/` folder.

1. `docdrift check` locally → report of broken claims with file:line on both sides.
2. GitHub Action fails PRs that break a documented command/path/signature without touching the doc.
3. PR bot comment: "this PR renames `createUser` → `registerUser`; 3 docs mention `createUser` (links)".
4. `docdrift annotate` interactive mode: helps add anchors to an existing doc corpus.

## 4. MVP scope & non-goals

**MVP:** claim extraction + deterministic checkers (shell commands vs package scripts/Make targets/binaries, file paths, CLI flags via `--help` schemas or clap/commander introspection where available, fenced code snippets that must compile/typecheck, symbol references vs tree-sitter index, internal links/anchors); explicit anchor comments; GitHub Action + SARIF output; PR-diff mode (only claims affected by the diff); LLM semantic warnings (optional, off in CI-gate mode).
**Non-goals:** generating/fixing docs (report only; token-furnace plan covers generation), hosted service (pure CLI/Action; no server), non-markdown formats (rst/asciidoc adapters later).

## 5. Tech stack

- **Core:** Rust (single fast static binary — this runs in every CI job, cold-start matters) with tree-sitter (md + top languages: TS/JS, Python, Go, Rust, Java, bash).
- **Snippet checking:** pluggable runners — `tsc --noEmit` for ts blocks, `python -m py_compile`, `go vet` on synthesized files; sandboxed, never executes snippet code, only parse/typecheck.
- **LLM layer (optional):** any OpenAI-compatible endpoint; used only in `--semantic` mode.
- **Distribution:** GitHub Action, Homebrew, cargo-binstall, pre-commit hook config.

## 6. How claims work

**Implicit claims (zero-config value):** extracted automatically from markdown — fenced `bash` blocks (each command word-0 checked against: package.json scripts, Makefile targets, repo binaries, allowlisted common tools), inline `code spans` that look like paths (checked exists) or symbols (checked against tree-sitter symbol index), relative links, fenced code in typed languages (typecheck).

**Explicit anchors (precision on demand):**
```md
<!-- docdrift: symbol=src/auth.ts#createUser -->
Call `createUser` with the email and role…
<!-- docdrift: snapshot=src/config.ts#DEFAULTS hash=a1b2c3 -->
```
Anchor kinds: `symbol` (exists + signature-hash stable), `snapshot` (doc block must match code block; `docdrift sync` updates hash after intentional change), `file`, `cmd`, `manual review-by=2026-12-01` (freshness timer).

**Semantic mode:** for prose paragraphs adjacent to anchors, LLM compares description vs current implementation → `warning: doc says retries 3 times; code retries 5`. Warnings only; each carries both citations.

## 7. Config (`.docdrift.toml`)

```toml
docs = ["README.md", "docs/**/*.md"]
code = ["src/**", "package.json", "Makefile"]
[checks]           # each: "error" | "warn" | "off"
commands = "error"
paths = "error"
symbols = "error"
snippets_typecheck = "warn"
links = "error"
semantic = "off"   # or "warn"
[ignore]
patterns = ["docs/archive/**"]
claims = ["cmd:legacy-deploy"]  # explicit suppressions, must carry a reason comment
```

## 8. Feature specs

**F1 — `docdrift check`.** Full-repo scan → human report (grouped by doc) + `--format sarif|json`; exit non-zero on errors. *AC:* zero false positives on a 10-repo real-world corpus for `commands|paths|links` at default config (the trust bar — curate the corpus in-repo as submodule fixtures); 200-file docs corpus scans < 5 s.

**F2 — PR-diff mode.** Given base..head: claims whose *code side* changed → re-check; symbols renamed/deleted in diff → find docs mentioning them. *AC:* rename detection catches `git diff` renames + tree-sitter symbol moves within a file; runtime < 10 s on fixture monorepo PRs.

**F3 — Anchors & sync.** Parse/validate anchor comments; `docdrift sync` interactively (or `--all`) re-hashes intentional changes; `annotate` suggests anchor placements for an unanchored corpus (heuristic: code spans + headings). *AC:* anchor round-trip — break a symbol, check fails; `sync`, check passes.

**F4 — GitHub Action + bot comment.** Action wraps check in PR-diff mode, uploads SARIF (inline annotations); optional sticky PR comment summarizing errors/warnings incl. semantic ones. *AC:* annotations land on the *doc* lines; comment edits itself; forks-without-secrets degrade gracefully (semantic off).

**F5 — Semantic warnings.** As §6, opt-in. *AC:* on a seeded fixture set (15 true drifts, 15 in-sync pairs): ≥ 12/15 caught, ≤ 2/15 false alarms; output always cites both sides.

**F6 — Snippet typecheck.** Synthesizes a compilable context per snippet (imports resolved against repo, `// docdrift-hidden` setup lines supported). *AC:* README snippet using a renamed export fails the check; never executes snippet code (enforced by runner design + test).

## 9. Milestones

- **M0 (1):** Rust CLI skeleton, md claim extraction, `commands|paths|links` checkers + report. *Ships: immediately useful.*
- **M1 (2):** symbol index + symbol checker + anchors + `sync`. 
- **M2 (3):** PR-diff mode + Action + SARIF + sticky comment. *Ships: the CI guard.*
- **M3 (4):** snippet typecheck runners (ts/py/go) + `annotate`.
- **M4 (5):** semantic mode + fixture corpus + false-positive telemetry (`--stats`), distribution polish. *Ships: v1.0.*

## 10. Testing

- False-positive corpus (real repos as fixtures) run in CI — any new FP fails the build; this is the product's core covenant.
- Golden reports per checker; property tests on the md extractor (nested fences, html blocks, tables).
- Action E2E on a fixture repo via workflow in this repo's CI.

## 11. Risks & open questions

- FP risk is existential (one wrong CI failure = uninstalled) → conservative defaults, curated allowlists, suppression with reasons, and the FP corpus gate.
- Command extraction ambiguity (prose-y code blocks) → only check fenced blocks tagged `bash|sh|console` and lines starting with `$ ` or a known runner (`npm|pnpm|make|cargo|go|uv`…); everything else ignored.
- Semantic mode cost/noise → warn-only forever by default; per-paragraph cache keyed on (prose hash, code hash).
