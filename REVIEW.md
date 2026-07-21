# Build Plan Library — Audit Review

## Overview

This review evaluates all 31 build plans against the standards in `plans/CONVENTIONS.md`. Each plan was assessed on three dimensions:
1. **Quality** — Structural completeness per conventions, clarity, and implementation specificity for AI agent execution
2. **Guardrails** — Whether scope boundaries (non-goals, MVP limits) are well-calibrated, too tight, or too loose
3. **Missing Details** — Critical or important information gaps in either PLAN.md or IMPLEMENTATION.md

**Methodology:** Each project's PLAN.md and IMPLEMENTATION.md were read in full. Claims in this review reference specific sections (§) of the actual files. Assessments were revised where initial characterization did not match file contents on close reading.

## Summary Table

| Project | Quality | Guardrails | Key Issues |
|---------|---------|-----------|-----------|
| agent-deck | Strong | Well-calibrated | None critical; minor pack versioning strategy gap |
| agent-loop | Strong | Tight/justified | Memory eval fixture corpus not linked; shell denylist not codified upfront |
| ai-code-reviewer | Strong | Well-calibrated | Golden benchmark corpus schema unspecified; diff-position algorithm not pseudocoded |
| depwatch | Adequate | Tight/sensible | API-diff sigHash normalization underspecified (how overloads merge into one hash); lockfile diff parsing lacks algorithm pseudocode; golden benchmark expected outputs referenced but not linked |
| doc-drift | Strong | Excellent | Symbol-matching heuristic not algorithmically specified; snippet synthesis light |
| dropbox-lite | Strong | Well-calibrated | None critical; Windows path normalization deferred (acceptable) |
| fitness-tracker | Strong | Appropriate | Progression deload logic ambiguous; e1RM bound and stall detector vague |
| flaky-detective | Strong | Conservative | `isFlakySignalWeighted` return type not formalized as a function signature; rename tracking similarity threshold (0.8) unjustified; golden benchmark corpus labels schema not described |
| foldersync-clone | Strong | Appropriate | rclone filter mapping not pseudocoded; bisync conflict mapping unclear; delete guard UX unspecified |
| landrop | Strong | Tight | Multi-subnet room segmentation algorithm light; guest policy enforcement mechanism described but not pseudocoded; secure-context fallback UX for WebRTC could be clearer |
| md-polish-skill | Strong | Well-calibrated | None critical |
| meal-tracker | Strong | Slightly loose | Repertoire scoring τ=0.86 threshold unjustified; vision model non-compliant output handling limited to 1 retry; calibration EMA α=0.3 lacks grounding |
| mock-studio | Strong | Well-calibrated | Response validation behavior on violation unclear; state inference ambiguity heuristic not formalized |
| molt-td | Strong | Excellent | Very minor: difficulty-tier multipliers only in IMPL not PLAN; visual feedback spec light |
| open-canvas | Adequate | Well-calibrated | Edit repair prompt template not committed; provider `fullRewriteOnly` threshold informal; intent classification fallback undefined |
| photo-memory | Strong | Well-calibrated | Entity extraction format unspecified; kind classification heuristic missing; masking UX flow unclear |
| plan-together | Strong | Well-calibrated | Vote deadline close behavior unspecified; ICS field spec missing; SSRF guard rules not inline |
| prompt-arena | Strong | Well-calibrated | Very minor: JS grader sandbox isolation detail; Elo K-factor unjustified |
| reddit-recall | Adequate | Slightly loose | "kmeans-lite" initialization/k-selection not detailed beyond ≤6 clusters; distillation map-reduce prompt structure not committed as fixture; digest permalink validation edge cases thin |
| reply-debt | Strong | Well-calibrated | LWW merge tie-break consistency; drift cadence algorithm lacks formal definition |
| repo-explainer | Strong | Well-calibrated | Framework heuristic registry logging format; Mermaid clustering algorithm when modules >14 |
| shift-roster | Strong | Well-calibrated | None critical — production-ready |
| skill-forge | Adequate | Well-calibrated | Rubric self-check pass criteria stated (≥5/6) but automated verification method unclear; brainstorm proposal frequency-evidence scoring is prose not algorithm; hallucinated-path prevention relies on scan-citation rule without enforcement test |
| tabsweep | Strong | Well-calibrated | Form-activity heuristic confidence unquantified; fake-time testing mechanism unelaborated |
| termius-clone | Strong | Well-calibrated | russh fallback trigger criteria vague; sync conflict resolution UI unsketched; jump-host depth limit unjustified |
| token-furnace | Strong | Excellent | Verify-pass regenerate reason format unspecified; spot-check threshold undefined; job module interface contract informal |
| unrecipe | Strong | Well-calibrated | Duplicate detection thresholds unjustified; ingredient merge order unspecified; timer background handling platform-specific |
| uptime-board | Strong | Well-calibrated | Assertion chain AND/OR semantics unclear; escalation failure recovery path missing; heartbeat start interaction unspecified |
| voice-inbox | Strong | Well-designed | Span overlap tolerance (10%) unjustified; merge hint trigram threshold (0.55) unjustified; webhook backoff base delay and cap unspecified |
| warranty-vault | Strong | Well-calibrated | Merchant lookup normalization undefined; PDF dossier embedding format unspecified; multi-item split stickiness unclear |
| weekly-rewind | Strong | Exemplary | Citation validator edge case (uncitable bullets); mask determinism for FTS; project matcher stickiness scope |

## Detailed Findings by Project

### agent-deck
**Quality: Strong** — Comprehensive vision with all required sections. WS protocol documented precisely, state engine pack schema well-defined, milestone task lists concrete with testable criteria (e.g., "added latency < 5 ms").

**Guardrails: Well-calibrated** — MVP scope (single-machine PTY + web UI + push + auth) is tight but justified. Multi-machine hub correctly deferred to v1.

**Missing Details: None critical** — Minor gap: pack versioning strategy (SemVer vs incremental) not specified.

---

### agent-loop
**Quality: Strong** — Exceptionally well-written with sharp positioning. LOC budget enforced by CI, loop contract pseudocoded, config formats exact (TOML schemas).

**Guardrails: Tight but justified** — The 3k-LOC budget is the product's covenant. Single-user trust focus prevents platform-building scope creep.

**Missing Details: Minor** — Memory eval fixture corpus path not committed; shell metacharacter denylist not codified upfront in §6 (only in milestone task); time parsing confirm-echo format unspecified.

---

### ai-code-reviewer
**Quality: Strong** — Two-stage verify pipeline, learning loop, and self-hosted model are well-specified. Pipeline contract pseudocode-detailed.

**Guardrails: Well-calibrated** — MVP ships single-lens reviewer with comment budget cap. Noise risk mitigated by verify stage and learning loop.

**Missing Details: Important but manageable** — Golden benchmark labels schema not specified (must_find binary or weighted?); config file field types not formally declared; diff-position mapping algorithm not pseudocoded despite being release-blocking.

---

### depwatch
**Quality: Adequate** — Pipeline steps and database schema are clear. Usage confidence tiers are well-defined (`static > require > reexport > dynamic(unknown)` in §0). Verdict join logic specified in §5 (symbol intersection with dotted-suffix fallback). Algorithmic specificity is lower than peers on some points.

**Guardrails: Tight/sensible** — npm-only MVP with PyPI deferred. Excludes auto-fix and vulnerability scanning.

**Missing Details: Moderate** — API-diff `sigHash` normalization undefined (how are overloads merged into one hash? IMPLEMENTATION §5 only says "normalized text of type"); lockfile diff parsing has no algorithm pseudocode (parsers listed for npm/pnpm/yarn-berry but behavior not specified); golden benchmark expected outputs referenced in `bench/bumps.yaml` but not linked to fixture repos.

---

### doc-drift
**Quality: Strong** — Sharp two-layer design (deterministic gates + LLM warnings). Anchor comment syntax exact, claim model pseudocode-defined, CLI contract precise.

**Guardrails: Excellent** — Semantic mode is warn-only. Report-only (no auto-fix). Non-markdown excluded.

**Missing Details: Minor** — Symbol-matching heuristic (edit distance? exact? dotted-path?) not algorithmically specified; snippet typecheck synthesis logic (import resolution) light; semantic cache key computation (prose hash granularity) undetailed.

---

### dropbox-lite
**Quality: Strong** — Journal-based sync protocol pseudocode-detailed with every operation specified. Conformance harness positioned before real client (excellent sequencing).

**Guardrails: Well-calibrated** — Server-canonical model justifies excluding E2E encryption and real-time collab.

**Missing Details: None critical** — Windows path normalization rules deferred to DECISIONS.md (acceptable for MVP).

---

### fitness-tracker
**Quality: Strong** — Progression rules specified by function contract. Domain-layer boundary enforced by eslint. Golden-table tests (≥40 cases per rule) anchored.

**Guardrails: Appropriate** — Local-first PWA with optional encrypted sync deferred to M4.

**Missing Details: Minor** — Progression deload ambiguity: "3 consecutive fails" unclear (sessions vs sets?), scope of −10% deload (exercise vs program?); e1RM ≤12 reps bound unjustified; stall detector "slope ≤ 0" computation method vague (linear regression? moving average?).

---

### flaky-detective
**Quality: Strong** — State machine explicit, detection engine signals listed with quantified weights (+3 pseudo-observations for strong signals, weak for cross-commit decorrelation), simulation test-bed positioned as crown jewel. Beta-Bernoulli posterior with exponential decay (half-life 14 days) specified in IMPLEMENTATION §0 and §6.

**Guardrails: Conservative by design** — Quarantine automation deferred to M4. PR comments only when ALL failures are confirmed flakes.

**Missing Details: Minor** — `isFlakySignalWeighted` referenced in §6 stats.go but not formalized as a named function signature with return type; rename tracking trigram similarity threshold (0.8) unjustified; golden benchmark corpus labels schema (what constitutes ground truth) not described.

---

### foldersync-clone
**Quality: Strong** — Run lifecycle pseudocoded (dry-run → guard → execute → parse → notify). Bisync first-run rule documented.

**Guardrails: Appropriate** — One-way mirror in MVP, two-way "beta" labeled. Wraps rclone as library (not shell-out).

**Missing Details: Moderate** — rclone filter rule conversion logic not pseudocoded; bisync `keep_both` → `none` mapping unclear; delete guard UX (warning? auto-abort? per-pair config?) not specified; OAuth credential storage mechanism vague.

---

### landrop
**Quality: Strong** — More thorough than initially assessed. P2P DataChannel framing specified in IMPLEMENTATION §4 (16-byte header: chunkIndex u64, len u32 + bytes; ACK every 16 chunks; flow control window 32). Relay backpressure explicit (429 + retry-after in §0). WS protocol comprehensive. Chunking, resume via bitmap, and BLAKE3 verification all detailed.

**Guardrails: Tight** — Relay as guaranteed floor, P2P as progressive enhancement. Zero-install web-based. TLS decision pragmatic (plain HTTP default, self-signed optional for WebRTC secure context).

**Missing Details: Minor** — Multi-subnet room segmentation mentioned but algorithm for interface detection light (just "room key = interface network"); guest policy enforcement mechanism described (cookie-based for PIN rooms) but not pseudocoded for the `guests-limited` filtering path; endianness of DataChannel frames not explicitly stated (likely platform-native but should be pinned).

---

### md-polish-skill
**Quality: Strong** — Precise conversion pipeline, CLI contracts exact, CSS assertion test format defined. All acceptance criteria map to concrete test types.

**Guardrails: Well-calibrated** — Appropriately tight for a skill (not a product). Single-file + combine mode, three themes, no hosting.

**Missing Details: None critical.**

---

### meal-tracker
**Quality: Strong** — More complete than initially assessed. Correction re-estimation clearly specified in IMPLEMENTATION §0: "local recompute for portion ops, LLM only for free-text swaps" with dedicated API route `POST /api/meals/{id}/correct-text`. Estimation flow (§5) includes full pipeline pseudocode with repertoire matching, confidence gating, and calibration multiplier application.

**Guardrails: Slightly loose** — MVP includes repertoire matching/learning which risks scope creep if accuracy benchmarks fail. AC "±25% for ≥70%" is empirically tuned without grounding.

**Missing Details: Minor** — Repertoire scoring τ=0.86 AND dHash ≤10 thresholds lack calibration methodology; vision model non-compliant output handling limited to "1 retry with error feedback" (no further degradation path); calibration EMA α=0.3 unjustified; correction bias capping [0.85,1.15] range unexplained.

---

### mock-studio
**Quality: Strong** — Request pipeline is a normative flow diagram, scenario YAML format fixed, CEL condition syntax locked, CLI contract explicit.

**Guardrails: Well-calibrated** — OpenAPI 3.0/3.1 only. No auth simulation, cloud mode, or contract testing.

**Missing Details: Minor** — Response validation violation behavior (pass-through or reject?) unspecified; state inference ambiguity detection heuristic not formalized; cold-start loading strategy (lazy vs full-load) not detailed.

---

### molt-td
**Quality: Strong** — Exceptionally rigorous for a game project. Evolution system mathematically specified, damage pipeline order explicit, content schemas locked (zod).

**Guardrails: Excellent** — Vertical slice MVP (1 biome, 6 towers, 5 damage types, 12 species). No multiplayer or level editor.

**Missing Details: Very minor** — Difficulty-tier multipliers only in IMPL not PLAN; visual/audio treatment for "resist ding" not pseudocoded.

---

### open-canvas
**Quality: Adequate** — Core loop and edit protocol clear. Aider-style edit block format specified normatively in IMPLEMENTATION §3 with apply ladder (exact → whitespace-fuzzy → context-anchor fuzzy → Babel AST fallback). Repair loop (max 2 retries) defined but prompt template not included.

**Guardrails: Well-calibrated** — Single-file artifacts only. No multi-file projects, image generation, or collaboration.

**Missing Details: Moderate** — Edit repair prompt template and "real context" definition not committed; provider `fullRewriteOnly` capability flag mentioned in §7 T4 as "capability auto-flag from bench results" but detection threshold not formalized; intent classification router has "heuristic first" fallback (make/create/build keywords) but its failure mode is undefined; renderer contract versioning mentions `v1/` directory but migration strategy unstated.

---

### photo-memory
**Quality: Strong** — Pipeline pseudocoded with degradation path. Hybrid retrieval with confidence gating for ask flow.

**Guardrails: Well-calibrated** — Privacy-first (encryption at rest, opt-in cloud). Excludes camera roll import and native apps.

**Missing Details: Important** — Entity extraction field formats (hex? name? RGB?) unspecified; kind classification decision tree/prompt missing; auto-collection centroid comparison timing unclear; secret masking UI flow (show "••••" or omit?) not specified.

---

### plan-together
**Quality: Strong** — Window-ranking algorithm pseudocoded, settlement algorithm property-tested, SSE event schema locked.

**Guardrails: Well-calibrated** — Date matcher + voting + itinerary + expense splitting. No payment processing (link out), no chat.

**Missing Details: Minor** — Vote deadline close behavior (voting disabled before or after tally?) unspecified; ICS property set (UID, DTSTART format, timezone handling) not detailed; SSRF guard rules not documented inline.

---

### prompt-arena
**Quality: Strong** — Project format locked with JSON schemas. Grader kinds enumerated. Blind A/B Elo with confidence requirements.

**Guardrails: Well-calibrated** — Single-turn MVP. Files + git as the collaboration tool (no hosted platform).

**Missing Details: Very minor** — JS grader `node:vm` context isolation specifics (eval access?); Elo K=32 choice unjustified.

---

### reddit-recall
**Quality: Adequate** — Value proposition clear. Distillation pipeline is specified (cluster → map → reduce) but "kmeans-lite" lacks initialization method and k-selection logic beyond "≤ 6 clusters." Comment pruning is deterministic and well-specified (score/depth rules in §0).

**Guardrails: Slightly loose** — Accuracy bar described empirically ("≥4/5 human rubric") without formal rubric schema committed as fixture.

**Missing Details: Moderate** — Clustering initialization/k-selection not detailed (how is k chosen? random seeding?); distillation map-reduce prompt structure described in concept but not committed as a versioned fixture file; auto-collection cosine ≥ 0.78 threshold unjustified; collection centroid recomputation timing ("nightly") means suggestions can be stale for active users.

---

### reply-debt
**Quality: Strong** — Scoring function locked, deep-link matrix per platform, sync protocol (field-level LWW + tombstones) specified.

**Guardrails: Well-calibrated** — Manual/share-based capture by design. No automated integrations or AI replies.

**Missing Details: Minor** — LWW tie-break (deviceId lexical) consistency across all paths; drift cadence selection algorithm lacks formal definition; screenshot encryption key management unspecified.

---

### repo-explainer
**Quality: Strong** — Six-stage pipeline with idempotent checkpoints. PageRank for ranking. Citation validator is release-blocking with own unit suite.

**Guardrails: Well-calibrated** — Excludes private-repo OAuth, IDE plugins, monorepos >500k LOC. No PR review or code editing.

**Missing Details: Minor** — Framework detection heuristic logging format undefined; Mermaid clustering algorithm when modules >14 not specified; citation validator logging content unclear.

---

### shift-roster
**Quality: Strong** — RLS policies and constraints named, tRPC surface comprehensive with role enforcement, DST suite as release gate. Production-ready specification.

**Guardrails: Well-calibrated** — Operational loop (roster → publish → confirm → clock → export) without payroll, award interpretation, or AI scheduling.

**Missing Details: None critical.**

---

### skill-forge
**Quality: Adequate** — Three modes described with concrete behavior specs. Rubric IS specified as the oracle (6 checks, ≥5/6 pass, description mandatory) in §0 and §3. Trigger overlap detection defined as "pairwise: do two descriptions claim the same trigger phrases?" in IMPLEMENTATION §2.

**Guardrails: Well-calibrated** — Brainstorm + build + audit in MVP. No marketplace or hook automation.

**Missing Details: Moderate** — Brainstorm ranking formula (`score = frequency-evidence × automation-absence`) is prose, not a computable algorithm; hallucinated-path prevention relies on "every proposal MUST cite a real path found in the scan" rule but no automated enforcement test is specified; rubric self-check is agent-run (the skill evaluates its own output) which is inherently limited; test protocol uses live session trigger tests (≥4/5) rather than deterministic unit tests.

---

### tabsweep
**Quality: Strong** — MV3 constraints acknowledged early. Dexie schema appropriate for IndexedDB. Permission audit (6, each justified).

**Guardrails: Well-calibrated** — Local-first, no cloud sync (export/import covers v1). Confirm-mode defaults and 30s undo prevent destructive mistakes.

**Missing Details: Minimal** — Form-activity heuristic confidence/false-positive rate unquantified; LLM embedding token cost unestimated; restore-scroll fallback for unsupported browsers unspecified; fake-time testing mechanism for MV3 alarms unelaborated.

---

### termius-clone
**Quality: Strong** — One of the most elaborate specs. IPC contract typed (Tauri commands — 20+ commands enumerated in IMPLEMENTATION §5), SQLite schema has canonical sync_state table, milestone sequencing sophisticated (M0–M5 with clear shipping gates).

**Guardrails: Well-calibrated** — Locally-functional without sync (M0–M3). Self-hosted sync server is key differentiator in M4–M5.

**Missing Details: Minor** — russh fallback IS specified in both PLAN §11 ("fallback is `libssh2` bindings") and IMPLEMENTATION §0 ("switch to `ssh2` crate for the agent path only") but the decision trigger ("if agent-forwarding blocks M2") is vague — what specifically constitutes "blocking"?; Windows key storage uses `keyring` crate (DPAPI on Windows, Keychain on macOS, Secret Service on Linux per §0) but integration with age-style key derivation not shown; sync conflict resolution is specified (LWW + conflict copies) but UI for resolving/merging conflicts is not sketched; jump-host depth limit (4) unjustified.

---

### token-furnace
**Quality: Strong** — Six fixed job types, deterministic analysis phase before LLM. Budget enforcement with resumption is sophisticated.

**Guardrails: Excellent** — No arbitrary agentic coding, no auto-merge. Jobs are templated pipelines. Deterministic pre-analysis prevents hallucination.

**Missing Details: Minor** — Verify-pass regenerate "reason" format not specified; spot-check reject-rate threshold and pause-then-notify flow undefined; `prompts` interface contract (object shape vs function?) informal; secret-redaction regex patterns not listed.

---

### unrecipe
**Quality: Strong** — Extraction ladder (JSON-LD → readability+LLM → vision) is elegant. Parser golden suite (150 real lines) anchors quality.

**Guardrails: Well-calibrated** — Capture + box + cook + pantry + shopping list. No social sharing, nutrition facts, or meal calendars.

**Missing Details: Minor** — Duplicate detection thresholds (0.7/0.6) unjustified; merged ingredient line order unspecified; share-link MVP fallback unclear; cook-mode offline persistence mechanism undefined; timer background handling platform limits undocumented.

---

### uptime-board
**Quality: Strong** — Single-binary Go, hysteresis defaults tabulated, rollup retention rules explicit, faultserver for testing.

**Guardrails: Well-calibrated** — Core monitoring without distributed probes or APM. SLO/report deferred to M4.

**Missing Details: Implementation ambiguities** — Assertion chain semantics (AND/OR? short-circuit?) unclear; escalation send-failure recovery path missing; heartbeat "start" pings interaction with grace window unspecified; rollup bucket-boundary assignment undefined; built-in autocert contradicts static binary story.

---

### voice-inbox
**Quality: Strong** — Pipeline comprehensive. Date resolution guardrails explicit (never LLM-resolved dates). Auto-file guardrails prevent wrong-date slippage. Merge hint IS specified (trigram ≥ 0.55 vs open items same type within 7 days in §4). Webhook specifies 5 retries expo backoff in §0.

**Guardrails: Well-designed** — Review-first default with optional auto-file upgrade. Excludes contact matching and meeting diarization.

**Missing Details: Moderate** — Span overlap tolerance (10%) unjustified with no merge algorithm for which item wins; merge hint trigram threshold (0.55) and 7-day window lack calibration; webhook backoff base delay and cap not specified (only "expo backoff" + 5 retries); multi-day ICS event dtend calculation unspecified.

---

### warranty-vault
**Quality: Strong** — Extraction schema with confidence per field, knowledge pack structure versioned, multi-item splitter ($10 threshold) concrete.

**Guardrails: Well-calibrated** — Capture + confirm in MVP. No claim automation or bank integrations.

**Missing Details: Knowledge management gaps** — Merchant name normalization rules undefined; receipt embedding format in PDF dossier unspecified; email quarantine timeout scope (per-email vs per-entry) unclear; multi-item price-edit split stickiness undefined; reminder cancellation cascade for duplicate products untested.

---

### weekly-rewind
**Quality: Strong** — Privacy-first with three modes. Deterministic facts rollup separate from LLM narrative synthesis. Citation validator rule explicit.

**Guardrails: Exemplary** — "Never a team surveillance tool" as covenant. No screenshots, keystroke tracking, or team dashboards.

**Missing Details: Subtle but important** — Citation validator fallback when LLM can't cite after retry (replace section with deterministic facts? drop?) unspecified; mask determinism for FTS index consistency undefined; project matcher stickiness scope (per-user? global?) unclear; time estimate bin boundaries (overlapping?) unspecified; browser history schema version coverage not enumerated.

---

## Cross-Cutting Themes

### Strengths Across the Library
1. **Consistent structure** — All 31 projects faithfully follow the two-document format with correct section ordering
2. **Scope discipline** — Non-goals are consistently used to reject obvious feature creep; MVPs are independently shippable
3. **Testable acceptance criteria** — Most features include quantified, automatable criteria
4. **Milestone sequencing** — Task lists are generally concrete enough for top-to-bottom agent execution
5. **Risk honesty** — Known unknowns are acknowledged rather than hidden

### Recurring Weaknesses
1. **Algorithm specification gaps** — Several projects describe algorithms in prose rather than pseudocode, creating ambiguity for implementing agents (depwatch lockfile parsing, reddit-recall kmeans-lite initialization, voice-inbox merge hints, open-canvas intent classification)
2. **Unjustified thresholds** — Empirically-tuned constants (similarity τ=0.86, decay half-lives, K-factors, overlap tolerances, cosine ≥0.78) are locked without justification or calibration methodology. These are the most common "@TUNE" candidates.
3. **Error recovery underspecification** — "Retry with error feedback" patterns lack specifics on trigger conditions, backoff strategies, max attempts, and fallback UX (voice-inbox webhook backoff, meal-tracker vision degradation beyond 1 retry)
4. **LLM prompt templates** — Projects using LLM calls describe intent but rarely commit prompt structure as versioned fixtures; an AI agent must invent the prompt at implementation time (open-canvas repair prompt, reddit-recall distillation map-reduce prompts)
5. **Edge-case semantics** — Boundary conditions (bucket edges, overlapping spans, concurrent merge tie-breaks, platform-specific limitations) are often left to idiom rather than explicit acceptance criteria

### Recommendations

1. **Add pseudocode for complex algorithms** — Anywhere the spec says "kmeans-lite", "trigram similarity ≥ X", or "decay with half-life" without showing the computation, add 5–15 lines of pseudocode. This is the single highest-impact improvement for AI agent executability.

2. **Justify or mark thresholds as TBD** — For every empirically-tuned constant, either cite a reference/rationale or explicitly mark it as `@TUNE` in IMPLEMENTATION §0 with a note that the milestone includes calibration against fixtures.

3. **Standardize error recovery spec** — Add a section to CONVENTIONS.md or each IMPLEMENTATION.md: "Error Recovery & Graceful Degradation" covering external API failures, malformed LLM outputs, timeout scenarios, and user-facing fallback UX.

4. **Commit prompt templates as fixtures** — For LLM-dependent projects, store representative prompts in `fixtures/prompts/` as versioned files referenced from IMPLEMENTATION.md. This eliminates prompt invention during implementation.

5. **Specify edge-case behavior in acceptance criteria** — Convert prose mentions of boundary conditions into explicit AC items (e.g., "When rollup bucket boundary coincides with a result timestamp, result belongs to the newer bucket").

## Tier Classification

**Production-Ready (agent can execute immediately):**
agent-deck, agent-loop, doc-drift, dropbox-lite, landrop, md-polish-skill, molt-td, plan-together, prompt-arena, reply-debt, shift-roster, token-furnace

**Minor Clarifications Needed (executable with DECISIONS.md as questions arise):**
ai-code-reviewer, flaky-detective, fitness-tracker, meal-tracker, mock-studio, repo-explainer, tabsweep, termius-clone, unrecipe, uptime-board, voice-inbox, warranty-vault, weekly-rewind

**Important Gaps to Address Before Execution:**
depwatch, foldersync-clone, open-canvas, photo-memory, reddit-recall, skill-forge

---

*Review conducted on branch `review/plan-audit`*
