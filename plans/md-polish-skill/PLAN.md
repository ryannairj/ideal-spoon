# MD Polish — a skill that turns any markdown into beautiful, non-slop HTML

**Category:** Claude Code skill · **Difficulty:** Easy · **Platform:** Claude Code (`.claude/skills/`)

## 1. Vision

We all drown in markdown — plans, reports, READMEs, meeting notes. `/md-polish path/to/file.md` turns any of them into a single self-contained HTML page that looks *designed*: real typography, purposeful hierarchy, working TOC, styled tables/code/callouts, light+dark themes. Explicitly anti-slop: no gradient-purple hero sections, no emoji confetti, no "✨ Overview ✨".

## 2. Why it can win

- The gap is taste. `pandoc -s` output looks like 2004; AI-generated pages look like AI. This skill encodes a **design system + hard rules** so output is consistently restrained and professional.
- Deterministic pipeline (converter + themes) with the LLM only making *semantic* enhancement decisions (callout detection, doc-type inference) → fast, cheap, reproducible.

## 3. Users & use cases

1. `/md-polish README.md` → `README.html`, self-contained, shareable.
2. `/md-polish docs/ --combine` → one multi-section page with sidebar nav from N files.
3. `/md-polish report.md --theme print` → print-optimized (PDF-ready via browser print).
4. Pipe-friendly: works on any .md the session produced (meeting notes, plans).

## 4. Scope & non-goals

**MVP:** single-file + combine mode; 3 themes (`doc`, `report`, `print`); TOC; syntax highlighting; mermaid rendering; callout transformation; local images inlined as data URIs; strictly self-contained output (zero external requests).
**Non-goals:** hosting/publishing (that's the Artifact tool's job — the skill can *offer* it), interactive widgets, editing content meaning, slide decks.

## 5. Deliverable structure

```
md-polish/
  SKILL.md                # workflow + design rules summary
  references/
    design-rules.md       # the anti-slop constitution (see F3)
    themes.md             # theme guide + when to pick which
  assets/
    convert.mjs           # node script: markdown-it + shiki + toc + inliner
    themes/doc.css  report.css  print.css
    template.html         # shell with TOC, theme toggle, print styles
```

Conversion is done by the bundled script (`node assets/convert.mjs in.md --theme doc`), NOT by the LLM writing HTML — the LLM's job is: pick theme, pass metadata (title/author/date if found), decide callout mappings, review output, and handle edge cases. This keeps 5,000-line markdown files cheap and deterministic.

## 6. Behavior spec

**F1 — Convert.** markdown-it (GFM: tables, task lists, footnotes, strikethrough) + shiki (dual light/dark syntax themes) + anchor links + generated TOC sidebar (collapsible on mobile) + mermaid blocks rendered via inlined mermaid.js only when the doc contains mermaid (else omitted to keep files small). *AC:* output opens from `file://` with zero network requests (CI checks no `http` refs in output); 1 MB markdown converts < 5 s.

**F2 — Semantic enhancement (LLM decisions, applied as flags to the script).** Doc-type inference (README/report/spec/notes → theme + layout defaults); blockquote-pattern → callout classes (`> **Note:**` → note box; warning/tip/danger variants); title/byline/date extraction to a proper header. *AC:* enhancement never rewrites body text — content hash of text nodes unchanged (verified by script assertion).

**F3 — The anti-slop constitution (design-rules.md, enforced in themes).** System font stack or one inlined open font subset; max line length 70ch; type scale 1.25; generous whitespace; muted single accent color; tables with hairline rules; no drop shadows on text, no gradients, no emoji injection, no decorative icons; dark mode via `prefers-color-scheme` + manual toggle; print stylesheet with page margins and repeated table headers. *AC:* themes pass a checklist test (grep-able CSS assertions: no `linear-gradient`, line-height ≥ 1.6, etc. — cheeky but effective).

**F4 — Combine mode.** N files → ordered sections (order: explicit arg > numeric prefix > README first > alpha), unified sidebar with per-file sections, per-section anchors, cross-file relative links rewritten to anchors. *AC:* internal links between combined files resolve within the page.

**F5 — Images & assets.** Local images inlined as data URIs (warn > 2 MB total, offer `--no-inline` with relative copies); remote images left as-is with a warning list (breaks self-containment — user's call). 

## 7. Milestones

- **M0:** convert.mjs with `doc` theme + template + TOC; SKILL.md happy path. *Ships: usable immediately.*
- **M1:** callouts, doc-type inference, `report` + `print` themes, image inlining.
- **M2:** combine mode, mermaid, checklist tests, polish + description tuning. *Ships: v1.*

## 8. Testing

- Golden fixture set: brutal markdown (nested lists 5 deep, giant tables, mixed HTML, footnotes, 3 languages of code, mermaid, broken links) → snapshot HTML + visual screenshot diff (Playwright) per theme, light+dark+print.
- Self-containment CI check; text-content-hash invariance check.
- Dogfood: this repo's plans rendered with it.

## 9. Risks & open questions

- Node availability in the session environment — script uses only `npm i` template with pinned deps, and SKILL.md includes a no-node fallback (pure-LLM conversion for small files, flagged as slower/costlier).
- Fonts: inlining a subset adds ~30–80 KB; default to system stack, font opt-in via `--font serif`.
