# Skill Forge — a Claude Code skill that brainstorms and scaffolds new skills

**Category:** Claude Code skill · **Difficulty:** Easy–Medium · **Platform:** Claude Code (`.claude/skills/`)

## 1. Vision

A meta-skill: run `/skill-forge` and it mines *your actual usage* — repo contents, recent session transcripts (if available), existing skills, CLAUDE.md — to propose skills you should have ("you've manually explained your release process 4 times; want a `/release` skill?"), then interviews you briefly and scaffolds a well-formed skill: `SKILL.md` with a triggering-optimized description, supporting reference files, and a test prompt set.

## 2. Why it can win

- Most users never write skills because the blank page is hard; the ideas are latent in their workflows. Mining + scaffolding removes both blockers.
- Output quality is enforceable: the skill embeds a rubric (description triggerability, progressive disclosure, no bloat) so generated skills follow best practice instead of dumping walls of prose.

## 3. Users & use cases

Claude Code users, plugin authors, teams standardizing workflows.

1. `/skill-forge` → 5 ranked skill proposals with evidence ("found deploy runbook in docs/, no deploy skill exists").
2. `/skill-forge build a skill for our PR-description house style` → interview (3–5 targeted questions) → scaffolded skill.
3. `/skill-forge audit` → review existing skills for triggering overlap, stale instructions, bloat.

## 4. Scope & non-goals

**MVP:** the three modes above; personal + project skill targets; generated test-prompt file per skill.
**Non-goals:** publishing to marketplaces, MCP server generation, hooks/settings automation (point to `update-config` instead), modifying skills without showing a diff.

## 5. Deliverable structure

This project *is* a skill package:

```
skill-forge/
  SKILL.md                 # entry: mode router + core workflow
  references/
    rubric.md              # quality rubric for generated skills
    description-guide.md   # how to write descriptions that trigger correctly
    patterns.md            # skill archetypes: workflow, checklist, generator, style-guide, integration
    interview.md           # question banks per archetype
  templates/
    SKILL.template.md
    test-prompts.template.md
```

## 6. Behavior spec

**F1 — Brainstorm mode (default).** Evidence scan: existing skills (dedupe), CLAUDE.md conventions, docs/runbooks, package scripts, CI workflows, README task sections. Output: table of ≤ 7 proposals — name, one-line purpose, archetype, evidence, effort (S/M/L) — ranked by (frequency of underlying task × absence of automation). Each proposal offers "build now". *AC:* every proposal cites concrete evidence found in the repo; no proposal duplicates an existing skill's trigger space.

**F2 — Build mode.** Pick archetype → ask only questions the scan couldn't answer (max 5, batched) → generate: SKILL.md (≤ 150 lines, progressive disclosure — details pushed to `references/`), description written per description-guide (what it does + when to trigger + trigger keywords), test-prompts file (5 should-trigger, 5 shouldn't-trigger prompts). Shows the full file tree as a diff before writing. *AC:* generated SKILL.md passes rubric self-check (the skill runs its own rubric and reports the score); files land in the user-chosen target (`~/.claude/skills/` vs `.claude/skills/`).

**F3 — Audit mode.** For each existing skill: description vs body mismatch, trigger overlap matrix with other skills, staleness signals (references to files that no longer exist), size bloat. Output: report + per-skill suggested edits (diff previews, apply on confirm). *AC:* never edits without explicit per-skill confirmation.

**F4 — Rubric (embedded, also the test oracle).** A generated skill must: have a description stating both *what* and *when*; fit the archetype pattern; keep SKILL.md lean with references split out; include concrete examples; avoid restating things Claude already does by default. 

## 7. Milestones

- **M0:** SKILL.md router + build mode with templates for 2 archetypes (workflow, style-guide); manual test pass.
- **M1:** brainstorm mode with evidence scan; remaining archetypes; test-prompts generation.
- **M2:** audit mode; rubric self-check; polish description for its own triggerability. *Ships: v1.*

## 8. Testing

- Fixture repos (one with runbooks/scripts, one bare) → brainstorm output snapshot review checklist.
- Trigger tests: run the generated test-prompt sets against a live session for 3 sample generated skills; should-trigger ≥ 4/5, shouldn't-trigger ≥ 4/5 abstention.
- Dogfood: use skill-forge to regenerate itself and diff against hand-written version.

## 9. Risks & open questions

- Session-transcript mining depends on what's accessible at runtime — treat as optional evidence source; repo mining alone must be sufficient.
- Over-eager triggering of skill-forge itself is ironic and bad — description must scope to explicit skill-creation intent.
