---
schedule: manual
enabled: true
template: true
title: Follow-ups
description: "Commitments and action items you took on today — so nothing slips"
icon: "✅"
featured: false
---

Scan my screen and audio recordings from today (last 16 hours only) for things
**I committed to** or **was asked to do**, and list the open follow-ups so I
don't drop any. This is read-only — do NOT write or patch anything, just report.

Read the screenpipe skill first so you know the search endpoints.

Look across BOTH sources, scoped to today:
- **Audio** (`content_type=audio`) — meetings and calls: things I said I'd do
  ("I'll send…", "I'll get back to you", "let me follow up on…", "I can have
  that by Friday"), and things others asked of me.
- **Screen/OCR + UI** (`content_type=ocr` / `content_type=ui`) — chats, email,
  Slack, Messages, issue trackers: action items assigned to me, "@me can you…",
  "could you…", unanswered questions directed at me, things I replied "yes" /
  "on it" / "will do" to.

For each follow-up, capture: **what** needs doing, **for whom** (person/channel),
**by when** if a due date or timeframe was mentioned, and the **source +
timestamp** so I can jump back to it.

Use this exact format:

## Open Follow-ups
- [ ] **<what>** — for <whom>{, due <when>} _(via <app> ~<time>)_
- [ ] ...

## Probably Already Done
- Items I committed to AND can see evidence I completed later in the day — with the timestamp of the evidence.

## Needs a Decision
- Things asked of me that I never acknowledged or answered — flag these first; they're the easiest to lose.

Rules:
- Only list items you can verify from the recordings — quote or reference the
  moment. Do not invent commitments.
- Deduplicate: if the same promise shows up in a meeting and a follow-up chat,
  list it once with both sources.
- If you find nothing actionable, say so plainly instead of padding the list.

End with: "**Most urgent:** [the one with the nearest deadline or the longest unanswered]".
