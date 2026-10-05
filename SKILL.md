---
name: onepress-podcast
description: Turn a topic, document, or slide deck into a narrated audio episode via OnePress — AI voices (including cloned voices), natural pacing, workspace delivery. Requires a free ONEPRESS_API_KEY; submits the task, polls, and reports where the MP3 lives.
version: 1.0.0
---

# OnePress Podcast

Produce audio content through [OnePress](https://www.getonepress.com) — podcast-style
episodes, narrated briefings, audio versions of documents or decks.

**This skill requires `ONEPRESS_API_KEY`** — audio synthesis runs on OnePress
infrastructure (TTS voices, mixing, delivery). There is no local mode; if the key
is missing, say so plainly and point the user to getonepress.com. Do not pretend
to generate audio locally.

## When to use

- "Turn this report into a podcast episode"
- "Make an audio version of my investor update / deck"
- "Record a briefing I can listen to on my commute"
- Voice-clone narration requests (user manages voices in the OnePress app)

## How it works

The key goes in the environment (`ONEPRESS_API_KEY`), created at
**getonepress.com → app → Settings → Account → API keys** (`opk_…`, shown once).

Submit a task describing the episode — topic or source material, target length,
tone, voice preference:

```
POST https://www.getonepress.com/api/v1/conversations
Authorization: Bearer $ONEPRESS_API_KEY
Content-Type: application/json

{"message": "Create a ~10-minute podcast episode on <topic>. Tone: <tone>. Use <voice notes>.", "title": "<title>"}
→ 202 {"conversationId":"conv_..."}
```

Poll until done (audio tasks take a few minutes):

```
GET https://www.getonepress.com/api/v1/conversations/conv_...
Authorization: Bearer $ONEPRESS_API_KEY

→ status "done": answer + preview_path (workspace-relative, e.g. "Audio/xxx.mp3")
```

Follow-ups (`POST` same conversation id) keep context — "make it shorter", "more energy".

MCP alternative: server `https://www.getonepress.com/api/mcp`, tools
`onepress_create_task` / `onepress_task_status` / `onepress_list_tasks`.

## Report back

- `answer` — the agent's summary
- `preview_path` — **a path, not a URL**; the MP3 lives in the user's OnePress
  workspace at https://www.getonepress.com/app (preview/download/share there)
- Conversation id/title

## Errors

401 bad key · 402 out of credits · 409 busy (keep polling) · 429 rate limited.

## Notes

- Be honest: audio is produced on getonepress.com, not here. Never fabricate files or links.
- Never ask the user to paste their API key into chat — env/config only.
