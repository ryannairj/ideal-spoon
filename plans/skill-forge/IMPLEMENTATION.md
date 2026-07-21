# Implementation Spec — Skill Forge

Read `PLAN.md` first. This file fixes the implementation decisions; do not re-decide them. The deliverable is a skill package (markdown + templates), not an app — so this spec pins file contents and structure.

## 0. Fixed decisions

| Decision | Value |
|---|---|
| Skill name / invocation | `skill-forge` → `/skill-forge [build …|audit]` |
| Install targets | offer both `~/.claude/skills/skill-forge/` (personal) and `.claude/skills/` (project); default personal |
| Archetypes (fixed set of 5) | `workflow` (multi-step procedure), `checklist` (review/verify pass), `generator` (produce an artifact from inputs), `style-guide` (how outputs should look/sound), `integration` (wrap a tool/CLI/API usage pattern) |
| SKILL.md size budget | ≤ 150 lines for generated skills; skill-forge's own SKILL.md ≤ 200 |
| Evidence sources (scan order) | existing skills → CLAUDE.md → `docs/**/*.md` (runbook-ish: headings matching deploy/release/setup/onboard/test/review) → `package.json` scripts / Makefile / justfile → `.github/workflows` → README task sections |
| Question budget | build-mode interview ≤ 5 questions, asked in ONE batch |
| Test prompts | every generated skill ships `test-prompts.md`: 5 should-trigger / 5 should-not |
| Rubric scoring | 6 checks, pass/fail each; generated skill must pass ≥ 5/6, description check mandatory |

## 1. Package layout (exact deliverable)

```
skill-forge/
  SKILL.md
  references/
    rubric.md
    description-guide.md
    patterns.md            # the 5 archetypes: structure template + example each
    interview.md           # question banks per archetype
  templates/
    SKILL.template.md
    test-prompts.template.md
```

## 2. SKILL.md content spec (write exactly this structure)

Frontmatter:
```yaml
---
name: skill-forge
description: Brainstorm, design, and scaffold new Claude Code skills. Use when the user
  wants to create a skill, asks what skills they should have, wants a workflow turned
  into a slash command, or wants existing skills audited for overlap/staleness/bloat.
  Triggers: "make a skill", "new skill", "/skill-forge", "audit my skills",
  "turn this into a skill". NOT for configuring settings/hooks (use update-config).
---
```
Body sections (in order):
1. **Mode router** — no args → Brainstorm; `build <topic>` or a described skill → Build; `audit` → Audit.
2. **Brainstorm mode** — the evidence scan procedure (ordered list of sources per §0 with what to extract from each: e.g. package scripts → "tasks run often but undocumented"); ranking formula (computable, see §5 for the scoring definition): `score = frequency_evidence × automation_absence`; output format: a markdown table `| # | Skill name | Archetype | One-liner | Evidence (path) | Effort |` capped at 7 rows, each row followed by nothing — end with "Reply with a number to build it."
   Hard rules: every proposal MUST cite a real path found in the scan (enforced by the path-citation check in §5); before proposing, list existing skills and drop overlaps.
3. **Build mode** — steps: (a) pick archetype from `references/patterns.md`; (b) gather answers from evidence FIRST, then ask remaining questions from `references/interview.md` in one AskUserQuestion batch (≤5, ≤4 options each); (c) generate files from `templates/`, honoring the ≤150-line budget by pushing detail into `references/` files of the new skill; (d) write description per `references/description-guide.md`; (e) generate test-prompts.md; (f) run rubric self-check from `references/rubric.md` and print the scorecard; (g) show the full proposed file tree + contents as a diff-style preview, ask target (personal/project), write files only after confirm.
4. **Audit mode** — per existing skill: read SKILL.md; checks: description-vs-body mismatch, trigger overlap (pairwise: do two descriptions claim the same trigger phrases?), staleness (referenced paths that no longer exist — verify with file checks), bloat (>200 lines or >30% restating default behavior). Output: report table + per-skill suggested edit as a diff preview; apply only on explicit per-skill confirmation.
5. **Never do** — edit skills without preview+confirm; propose skills duplicating built-ins; exceed question budget.

## 3. references/ content specs

**rubric.md** — the 6 checks, each with pass criteria + a good/bad example: (1) description states *what* AND *when* with trigger keywords; (2) archetype structure followed; (3) SKILL.md ≤ 150 lines, details in references/; (4) contains ≥ 1 concrete worked example; (5) no restating default Claude behavior; (6) test-prompts file present and plausible.
**description-guide.md** — formula: `<verb phrase capability>. Use when <situations>. Triggers: <quoted phrases>. NOT for <adjacent-but-wrong cases>`; 5 worked examples (good/bad pairs); note on third-person, no marketing adjectives.
**patterns.md** — per archetype: skeleton section list, 1 full miniature example skill (≤30 lines), "choose this when" guidance.
**interview.md** — per archetype 6–10 candidate questions with option sets, marked which are answerable-from-evidence (skip if found).

## 4. templates/ content specs

`SKILL.template.md`: frontmatter placeholders `{{name}}, {{description}}`; body scaffold per archetype with `{{sections}}` slots and HTML comments instructing the generator what goes where (comments stripped on generation). `test-prompts.template.md`: two headed lists with 5 slots each + instructions line "run these against a fresh session; expect ≥4/5 each side".

## 5. Build order & tests

**Ranking score (computable definition).** Each brainstorm candidate is scored `score = frequency_evidence × automation_absence`, both normalized to 0–1 so the product ranks proposals:

```
frequency_evidence = clamp01(
    0.4 * (task appears in package/Makefile/just scripts ? 1 : 0)
  + 0.3 * (task named in a docs/** runbook heading ? 1 : 0)
  + 0.2 * (task appears in a CI workflow ? 1 : 0)
  + 0.1 * (mentioned in README task section ? 1 : 0))

automation_absence = 1 - existing_coverage        # existing_coverage in {0, 0.5, 1}:
  # 1.0  an existing skill already covers this task     → absence 0  (drop as overlap)
  # 0.5  a script/alias exists but no skill wraps it     → absence 0.5
  # 0.0  no automation of any kind found                 → absence 1.0
```

Candidates with `automation_absence == 0` are dropped (overlap rule); the table shows the top 7 by `score` desc. This makes ranking reproducible on a fixture repo rather than a matter of taste.

**Path-citation enforcement (deterministic, testable).** Every proposal's `Evidence (path)` is validated before output: the generator resolves each cited path with a file check and drops (does not merely flag) any proposal whose path does not exist. This is exercised by a real test — `fixtures/rich-repo/` has an `expected-proposals.md` whose every cited path must resolve, and a negative case asserts that a fabricated path is filtered out. This converts the "cite a real path" rule from an honor system into a checkable gate.

**Degradation when evidence is thin.** On a bare repo (`fixtures/bare-repo/`, no scripts/runbooks): brainstorm emits at most 3 archetype-default proposals explicitly labeled "(no repo evidence — generic suggestion)" and never fabricates a citing path; if zero evidence exists it says so and asks the user what they want to automate instead of inventing proposals.

**M0** — T1 write SKILL.md (router + build mode) + patterns.md (workflow, style-guide only) + both templates; T2 manual pass: build a `pr-description` style-guide skill end-to-end in a fixture repo; iterate until rubric self-check passes.
**M1** — T1 brainstorm mode + evidence scan rules + scoring definition above; T2 remaining 3 archetypes + interview.md; T3 description-guide.md + rubric.md finalized; T4 fixture repos (`fixtures/rich-repo/` with runbooks+scripts, `fixtures/bare-repo/`) + expected-proposal checklists + path-citation check (review doc, since skills aren't unit-testable in the classic sense: a `TESTING.md` protocol with expected outcomes per fixture, plus the deterministic path-existence assertion).
**M2** — T1 audit mode; T2 trigger testing protocol: generate 3 sample skills, run their test-prompt sets in live sessions, record ≥4/5 both directions in TESTING.md results table; T3 dogfood: `/skill-forge build a skill for writing IMPLEMENTATION.md specs` and review output quality; T4 tune skill-forge's own description against accidental-trigger cases ("help me with this skill I'm building in Python" must NOT trigger).

## 6. Acceptance mapping

PLAN F1: proposals-cite-evidence rule is in SKILL.md hard rules + TESTING.md fixture check. F2: rubric scorecard printed every build; file-write-after-confirm rule. F3: per-skill confirm rule. F4: rubric.md is the oracle. Size budgets are stated in SKILL.md so the generating agent self-enforces.
