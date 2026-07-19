# Voice Inbox — ramble a voice note, receive structured life admin

**Category:** Life improvement · **Difficulty:** Medium · **Platform:** Mobile-first PWA (+ Telegram bot capture)

## 1. Vision

Your best thoughts arrive while walking, driving, doing dishes — and die there, because capturing them means typing into the right app with the right structure. Voice Inbox is one button: talk for 20 seconds (or 5 minutes, rambling freely), and it comes out the other side as *structured items* — tasks with due dates, calendar-ready events, notes filed by topic, shopping-list items, "tell X about Y" reminders — sitting in a review inbox you clear with taps. Capture at the speed of thought; organization happens without you.

## 2. Why it can win

- Voice memo apps give you an *audio graveyard*; assistants (Siri/Gemini) handle single-intent commands but choke on "also, three things…" rambles. The wedge: **multi-item extraction from unstructured rambles** + a trust-building review inbox (nothing enters your lists without a glance-and-tap confirm — until you *choose* auto-file for high-confidence items).
- Destination-agnostic: items can stay in-app or push out (ICS/CalDAV for events, webhook/Todoist/Things-URL for tasks) — it's an inbox, not another silo.

## 3. Users & use cases

Parents, ADHD workflows, commuters, anyone whose hands are busy when their brain is not.

1. School run: "gotta book the dentist for Ivy sometime next week, oh and we're out of dishwasher tablets, and tell Mum the flight lands at 6 not 5" → 1 task (due next week), 1 shopping item, 1 tell-reminder.
2. Post-meeting walk: 3-minute debrief ramble → meeting note (topics auto-headed) + 4 action items.
3. Driving: CarPlay-less capture via Telegram voice message to the bot → same pipeline.
4. Weekly: search "what did I say about the kitchen reno?" → transcript hits + filed notes.

## 4. MVP scope & non-goals

**MVP:** one-tap record PWA (offline-queue audio), Telegram bot capture, pipeline (transcribe → segment → classify → extract fields), review inbox (confirm/edit/merge/discard per item), destinations (internal lists: tasks/shopping/notes/reminders; ICS feed for events; generic webhook), search (transcripts + items), audio retention settings, auto-file rules for high-confidence recurring patterns.
**Non-goals:** being a full todo app (internal lists are deliberately minimal; the product is the *funnel* — push to real tools via integrations), real-time assistant conversation, meeting-recording/diarization of *other people* (single-speaker capture; recording others has consent implications — documented stance), wake-word/always-listening, native apps (PWA + Telegram covers capture surfaces).

## 5. Tech stack

- **App:** Next.js 15 PWA + TypeScript; MediaRecorder capture (opus), IndexedDB offline outbox; Web Push.
- **Backend:** Postgres (Drizzle) + S3 (audio) + BullMQ workers.
- **Transcription:** provider-agnostic STT interface — hosted Whisper-class API default; self-host mode runs `faster-whisper` container. Word-level timestamps kept.
- **Extraction:** LLM structured-output pass: transcript → `items[{type{task,event,note,shopping,reminder,tell},title, details, who?, due_raw?, due_resolved?, confidence, span[start_word,end_word]}]` — every item cites its transcript span (tap item → hear that bit). Date resolution deterministic post-pass (`chrono-node` on `due_raw` against user tz + "said_at").
- **Bot:** grammY (Telegram) → same ingestion endpoint.

## 6. Architecture

```
capture (PWA / Telegram) → audio blob → transcribe worker → transcript(words+ts)
→ extract worker → items(draft, cited spans, confidences) → review inbox
→ confirm: route to destination (internal list | ICS materialize | webhook)  
auto-file rules: (type + pattern + confidence ≥ τ) → skip inbox, notify digest-style
```

## 7. Data model

- `users(id, email, tz, telegram_chat_id, settings_json /*retention, auto-file, destinations*/)`
- `captures(id, user_id, source{pwa,telegram}, audio_ref, duration_s, said_at, status{queued,transcribing,extracting,review,done,failed}, transcript_json)`
- `items(id, capture_id, type, title, details, due_at, who, span_json, confidence, state{draft,confirmed,auto_filed,discarded,merged}, destination{internal,ics,webhook}, destination_ref, created_at)`
- `lists(user_id, kind{tasks,shopping,notes,reminders}, items → internal_entries(id, item_id, content, done, due_at, topic))`
- `rules(id, user_id, match_json /*type, keyword, person*/, action{auto_file,destination,topic}, hits)`
- `events_ics(user_id, uid, item_id, dtstart, summary)` — served as a private ICS feed URL.

## 8. Feature specs

**F1 — Capture.** PWA: giant record button, live waveform + elapsed, pause/resume, works from lock-screen-adjacent PWA shortcut; offline queue with visible pending count; Telegram: voice/audio/text messages accepted, replies with item summary when processed. *AC:* record→queued < 1 s from cold PWA open; airplane-mode capture uploads later intact; 10-min ramble accepted (chunked upload).

**F2 — Pipeline.** As §6. *AC:* fixture corpus of 40 scripted rambles (single + multi-intent, accents, kid-screaming-background, mixed "um" density): item segmentation F1 ≥ 0.85 vs labels; every item's span plays back the right audio slice; date resolution golden suite ("next Friday", "the 3rd", "tomorrow arvo" w/ tz) — wrong-date is the cardinal sin, so `due_resolved` below confidence shows as "when?" chip instead of guessing silently.

**F3 — Review inbox.** Cards grouped by capture; per-item: type chip (tap to re-type), title editable inline, due chip, confirm/discard swipe; "confirm all" per capture; merge duplicates hint (same-ish title within 7 d). *AC:* clearing a 5-item capture ≤ 6 taps; discarded items train nothing silently (explicit "why?" optional tags feed rules suggestions); inbox zero state is satisfying (streak-free, just clean).

**F4 — Destinations.** Internal lists (minimal, checkable, due-sorted); ICS feed (subscribe once in Google/Apple Calendar — events appear on confirm); generic webhook (JSON POST per confirmed item, HMAC-signed) enabling Todoist/Things/Notion via user-side glue (recipes documented); per-type default destination. *AC:* ICS feed validates + updates within calendar-app polling norms; webhook retries with backoff and surfaces dead endpoints in settings.

**F5 — Auto-file & rules.** Suggested after patterns ("you've confirmed 'shopping: item' 10× — auto-file shopping items ≥ 0.9 confidence?"); auto-filed items appear in a daily digest note for audit; one tap reverts + disables rule. *AC:* auto-file never applies to items with unresolved dates or `tell/reminder` types (guardrail table tested); revert restores item to inbox intact.

**F6 — Search & retention.** FTS over transcripts + items; play-from-hit; retention: audio auto-deletes after N days (default 30, "keep transcript forever" default on) or "delete audio on confirm" mode. *AC:* retention sweeper leaves no orphan blobs; search hit → correct audio offset ± 2 s while audio retained.

## 9. Milestones

- **M0 (1):** PWA capture + upload + transcription + raw transcript list. *Ships: voice memos with search — already useful.*
- **M1 (2):** extraction pipeline + review inbox + internal lists. *Ships: the core magic.*
- **M2 (3):** date resolution hardening + spans playback + Telegram bot.
- **M3 (4):** destinations (ICS, webhook) + per-type routing.
- **M4 (5):** auto-file rules + digests + retention/self-host STT compose + polish. *Ships: v1.0.*

## 10. Testing

- The 40-ramble labeled corpus (record once with varied voices/noise; store in repo LFS) is the permanent gate for F2 metrics — recorded STT/LLM fixtures in CI, nightly live with drift report.
- Date-resolution golden suite across tz/DST; span-alignment property tests (items' spans tile within transcript bounds, no overlaps > 10%).
- E2E: Playwright mobile capture → inbox → confirm → ICS/webhook fixtures receive; Telegram flow via bot test harness.
- Privacy: retention sweeps, HMAC verification, audio access authz.

## 11. Risks & open questions

- STT quality on names ("tell Mum" vs "tell Tom") → who-fields flagged low-confidence get a "who?" chip; contact-list matching is v1.1 (permission-heavy).
- Trust ramp: one wrong auto-filed date kills confidence → guardrails in F5 + review-first default are non-negotiable product law.
- Telegram dependency for hands-free capture → it's a bonus surface, not the core; PWA button remains the primary path; evaluate iOS Shortcuts/Action-button recipe as a doc-only addition.
