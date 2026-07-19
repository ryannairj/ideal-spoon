# Implementation Spec — MD Polish (skill)

Read `PLAN.md` first. This file fixes the implementation decisions; do not re-decide them.

## 0. Fixed decisions

| Decision | Value |
|---|---|
| Skill name | `md-polish` → `/md-polish <file(s)|dir> [--theme doc|report|print] [--combine] [--font serif] [--no-inline]` |
| Converter | Node script `assets/convert.mjs`, ESM, deps pinned: `markdown-it@14` (+ plugins `markdown-it-anchor`, `markdown-it-footnote`, `markdown-it-task-lists`), `shiki@1` (themes: `github-light`/`github-dark`), `mermaid@11` vendored min build inlined only when needed |
| Output | sibling `<input-stem>.html`, self-contained, zero network |
| Themes | `doc` (default: sidebar TOC), `report` (centered, title block, numbered headings), `print` (print-first, page margins, no sidebar) |
| Type system | system font stack default; `--font serif` inlines a subset of Source Serif (woff2 base64, subset latin) stored in `assets/fonts/` |
| Callout syntax | blockquotes starting `**Note:**`, `**Tip:**`, `**Warning:**`, `**Danger:**` (case-insensitive) → `<aside class="callout callout-note">` etc. |
| Combine ordering | explicit CLI order > numeric filename prefix > README.md first > alphabetical |
| LLM's role | ONLY: infer doc type → theme default; extract title/author/date to `--meta` JSON; decide `--font`; review output warnings. Never rewrites body text |
| Text invariance | convert.mjs computes SHA-256 of concatenated text nodes pre/post enhancement; mismatch = hard error |
| Node fallback | if `node` unavailable: LLM converts directly ONLY for files < 300 lines, using the same theme CSS pasted from assets; larger files → instruct user to install node |

## 1. Package layout

```
md-polish/
  SKILL.md
  references/{design-rules.md, themes.md}
  assets/
    convert.mjs  package.json (pinned deps)  template.html
    themes/{doc.css, report.css, print.css, base.css}
    fonts/source-serif-subset.woff2.b64
  fixtures/{brutal.md, combined/{01-intro.md, 02-api.md, README.md}, images/}
  test/{convert.test.mjs, selfcontained.test.mjs, visual.spec.ts (playwright)}
```

## 2. SKILL.md content spec

Frontmatter description: "Convert markdown files into beautiful, self-contained HTML pages (typography, TOC, dark mode, print-ready). Use when the user wants a markdown file rendered, prettified, exported, or shared as HTML/PDF-ready output. Triggers: '/md-polish', 'make this markdown look good', 'render this md as a page', 'export to html'. NOT for editing content or publishing to a URL (offer the Artifact tool for hosting)."

Body workflow: 1) resolve inputs (file/dir/glob; dir → offer `--combine`); 2) run `npm install --prefix assets` once if `node_modules` missing, then `node assets/convert.mjs <files> --theme <t> --meta '<json>'`; 3) LLM decisions per §0 (theme by doc-type heuristics listed inline: README→doc, anything with "report/analysis/plan" title or date → report; `--theme print` only on request); 4) read script warnings (oversized images, remote images, broken links) and relay with suggested flags; 5) offer: open locally, or publish via Artifact tool. Edge cases section: no-node fallback rule, huge files (>1 MB → warn about conversion time), non-UTF8.

## 3. convert.mjs behavior contract

CLI: `convert.mjs <files…> [--theme doc] [--combine] [--out path] [--meta json] [--font serif] [--no-inline]`.
Pipeline: parse (markdown-it + plugins, GFM tables/strikethrough on) → callout transform (per §0 syntax) → heading anchors + TOC tree (h2–h4) → shiki dual-theme highlight (CSS vars switch) → mermaid: if ```` ```mermaid ```` present, inline vendored lib + init script, else omit → images: local → data URI (warn if total > 2 MB; `--no-inline` copies to `<stem>_files/`), remote → keep + warning list → cross-file link rewrite in combine mode (`02-api.md#auth` → `#02-api--auth`) → inject into `template.html` (slots: title, meta block, toc, content, theme css, base css, toggle script) → write; exit code 0 with JSON warnings on stdout `{warnings:[…], outPath, textHash}`.
Template: semantic HTML, `<nav class="toc">` collapsible ≤ 768 px, dark-mode via `prefers-color-scheme` + manual toggle persisting to localStorage, print stylesheet always included (`@page` margins 2 cm, `thead {display: table-header-group}`).

## 4. design-rules.md (the anti-slop constitution — enforce via CSS assertions)

Rules with testable form: line length `max-width: 70ch`; `line-height ≥ 1.6`; type scale 1.25 (h1 2.44 rem … h4 1.0 rem); single accent color var `--accent` (default `#3b5bdb`), used only for links/active TOC; tables: 1 px hairlines, no zebra by default; code blocks: 0.9 em, soft background, no border-radius > 6 px; **banned** (grep-asserted in tests): `linear-gradient`, `text-shadow`, emoji in generated chrome, more than 2 font families, `!important` outside print overrides.

## 5. Build order & tests

**M0** — T1 convert.mjs core (parse→template→write) + doc theme + base.css + TOC; T2 SKILL.md happy path; T3 `brutal.md` fixture (nested lists ×5, giant table, inline HTML, footnotes, 3 code langs, unicode, broken links) + snapshot test; T4 self-containment test (regex: no `http(s)://` in src/href except explicit remote-image warns; opens via `file://` in playwright with network blocked → zero requests).
**M1** — T1 callouts + meta block + report/print themes; T2 image inlining + warnings; T3 font subset flag; T4 text-hash invariance assertion + test; T5 CSS-assertion test suite per §4.
**M2** — T1 combine mode (ordering rules, unified TOC, cross-links) + fixtures; T2 mermaid conditional inlining + fixture; T3 visual screenshot tests (playwright: 3 themes × light/dark/print-emulation on brutal.md, committed baselines); T4 dogfood: render this plans repo; T5 description tuning + no-node fallback text finalized.

## 6. Acceptance mapping

PLAN F1 → self-containment + speed test (1 MB < 5 s, timed in CI); F2 → text-hash test; F3 → CSS assertions; F4 → combine fixtures; F5 → image warning tests. TESTING.md documents the manual trigger-phrase protocol (5 should / 5 shouldn't).
