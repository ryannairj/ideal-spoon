# Project Ideas — Build Plan Library

A collection of **31 detailed build plans**, each in its own folder under `plans/`. Every plan is written to be picked up and implemented end-to-end by an AI coding agent (or a human) without needing to come back for clarification: vision, differentiation, tech stack, architecture, data model, feature specs with acceptance criteria, milestone-by-milestone task breakdown, and testing strategy.

Shared engineering standards that apply to every plan live in [`plans/CONVENTIONS.md`](plans/CONVENTIONS.md) — read that first before implementing any plan.

## Index

### Dev tools

| Plan | One-liner | Difficulty |
|---|---|---|
| [agent-deck](plans/agent-deck/PLAN.md) | Mission control for AI CLI agents (Claude Code, Codex, OpenCode, …) with browser + mobile view | Hard |
| [repo-explainer](plans/repo-explainer/PLAN.md) | Paste a GitHub repo, get an interactive guided tour and architecture explanation | Medium |
| [ai-code-reviewer](plans/ai-code-reviewer/PLAN.md) | CodeRabbit-style AI pull-request reviewer you can self-host | Hard |
| [depwatch](plans/depwatch/PLAN.md) | Dependency upgrade advisor: changelog diffs, risk scores, "is this bump safe?" | Medium |
| [flaky-detective](plans/flaky-detective/PLAN.md) | Flaky-test tracker that ingests CI results, ranks flakiness, and auto-quarantines | Medium |
| [doc-drift](plans/doc-drift/PLAN.md) | CI tool that detects documentation drifting out of sync with the code it describes | Medium |
| [mock-studio](plans/mock-studio/PLAN.md) | Instant API mock server + fixture studio generated from an OpenAPI spec | Medium |
| [uptime-board](plans/uptime-board/PLAN.md) | Self-hosted uptime monitor with a public status page, single binary | Easy–Medium |
| [prompt-arena](plans/prompt-arena/PLAN.md) | Local-first prompt/eval playground that compares LLMs side by side with graded runs | Medium |
| [token-furnace](plans/token-furnace/PLAN.md) | Puts cheap LLM tokens to work overnight: background doc generation & repo janitor jobs | Medium |
| [agent-loop](plans/agent-loop/PLAN.md) | A minimal, auditable personal agent loop (the un-bloated openclaw/nanoclaw) | Medium |

### Claude Code skills

| Plan | One-liner | Difficulty |
|---|---|---|
| [skill-forge](plans/skill-forge/PLAN.md) | A skill that brainstorms, drafts, and scaffolds *new* skills from observed workflows | Easy–Medium |
| [md-polish-skill](plans/md-polish-skill/PLAN.md) | Skill that turns any markdown into a beautiful, non-AI-slop, self-contained HTML page | Easy |

### Clones (done right)

| Plan | One-liner | Difficulty |
|---|---|---|
| [termius-clone](plans/termius-clone/PLAN.md) | Cross-platform SSH client with synced hosts, snippets, and SFTP (Termius clone) | Hard |
| [foldersync-clone](plans/foldersync-clone/PLAN.md) | Rule-based folder sync/backup engine with a scheduler UI (FolderSync Pro clone) | Medium |
| [dropbox-lite](plans/dropbox-lite/PLAN.md) | Self-hosted, lightweight Dropbox: single binary server + sync clients | Hard |
| [open-canvas](plans/open-canvas/PLAN.md) | Claude-style artifacts/design studio powered by any model (DeepSeek, GLM, Kimi, …) | Medium–Hard |

### SaaS / product

| Plan | One-liner | Difficulty |
|---|---|---|
| [shift-roster](plans/shift-roster/PLAN.md) | Rostering + timesheets for small shift-based businesses | Medium–Hard |
| [warranty-vault](plans/warranty-vault/PLAN.md) | Snap a receipt → tracked warranties, return windows, and expiry reminders | Medium |
| [reddit-recall](plans/reddit-recall/PLAN.md) | Turns your passive Reddit saves into digests and nudges you to actually read them | Medium |

### Life improvement

| Plan | One-liner | Difficulty |
|---|---|---|
| [photo-memory](plans/photo-memory/PLAN.md) | Remember anything by photographing it: OCR + vision + natural-language recall | Medium |
| [fitness-tracker](plans/fitness-tracker/PLAN.md) | Local-first workout logging with progressive-overload intelligence | Medium |
| [meal-tracker](plans/meal-tracker/PLAN.md) | Photo-first meal logging with LLM nutrition estimation, zero database tedium | Medium |
| [unrecipe](plans/unrecipe/PLAN.md) | Recipe de-blogger: clean recipes, pantry awareness, one-tap shopping lists | Medium |
| [weekly-rewind](plans/weekly-rewind/PLAN.md) | Auto-journal that writes your week from git, calendar, and activity signals | Medium |
| [reply-debt](plans/reply-debt/PLAN.md) | Personal CRM that tracks who you owe replies to before friendships rot | Medium |
| [voice-inbox](plans/voice-inbox/PLAN.md) | Ramble a voice note, get structured tasks/notes/events in your inbox | Medium |
| [tabsweep](plans/tabsweep/PLAN.md) | Browser-tab hoarder rehab: triage, decay, and weekly digest of your 400 open tabs | Easy–Medium |

### Social

| Plan | One-liner | Difficulty |
|---|---|---|
| [plan-together](plans/plan-together/PLAN.md) | Plan trips & events with friends: polls, itinerary, costs — without the group-chat chaos | Medium–Hard |
| [landrop](plans/landrop/PLAN.md) | AirDrop for everything on your LAN: files, text, clipboard — zero install, just a URL | Easy–Medium |

### Games

| Plan | One-liner | Difficulty |
|---|---|---|
| [molt-td](plans/molt-td/PLAN.md) | Roguelike tower defense where enemies *evolve resistances* to the damage you overuse | Medium–Hard |

## How to implement a plan

1. Read `plans/CONVENTIONS.md`.
2. Read the chosen plan's `PLAN.md` fully before writing code.
3. Work milestone by milestone (M0, M1, …); each milestone is independently shippable and has acceptance criteria.
4. Do not expand scope beyond the plan's "Non-goals" without human sign-off.
