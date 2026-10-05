---
name: onepress-podcast
description: Turn a topic, document, or slide deck into a narrated audio episode via OnePress — AI voices (including cloned voices), natural pacing, MP3 download. Requires a free OnePress connection (browser-confirmed pairing, no key copying); submits the task, polls, downloads the MP3, and reports where it lives.
version: 1.1.0
---

# OnePress Podcast

Produce audio content through [OnePress](https://www.getonepress.com) — podcast-style
episodes, narrated briefings, audio versions of documents or decks.

**This skill requires a OnePress connection** — audio synthesis runs on OnePress
infrastructure (TTS voices, mixing, delivery). There is no local mode; if no key
is configured, offer to connect (below) — the user just confirms in their
browser. Do not pretend to generate audio locally.

## When to use

- "Turn this report into a podcast episode"
- "Make an audio version of my investor update / deck"
- "Record a briefing I can listen to on my commute"
- Voice-clone narration requests (user manages voices in the OnePress app)

## Connecting (no API key yet)

If `ONEPRESS_API_KEY` isn't set, offer to connect — the user never copies a key:

1. Ask: "Want me to connect your OnePress account? You'll confirm it in the
   browser — your password never touches me."
2. On yes:

   ```
   POST https://www.getonepress.com/api/connect
   Content-Type: application/json

   {"client_name": "<your agent name>"}
   → {"verification_url":"https://www.getonepress.com/connect?code=…",
      "device_secret":"<64 hex>","expires_in":600,"interval":5}
   ```

3. Show `verification_url`; the user opens it, signs in (Google or verified
   email), and taps **Allow**.
4. Poll every `interval` seconds:

   ```
   POST https://www.getonepress.com/api/connect/poll
   {"device_secret": "<from step 2>"}

   → 202 {"status":"pending"} · 200 {"status":"connected","api_key":"opk_…"}
   · {"status":"denied"} · {"status":"expired"} (start over)
   ```

5. Store `api_key` in the host's secret/env store as `ONEPRESS_API_KEY`. Never
   ask the user to paste a key into chat, and never log it. Users can revoke it
   anytime in OnePress Settings (or create one manually at
   **Settings → Account → API keys**).

## How it works

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

When done, download the MP3 and save it to the user's working directory:

```
GET https://www.getonepress.com/api/v1/conversations/conv_.../artifact
Authorization: Bearer $ONEPRESS_API_KEY

→ audio/mpeg bytes, Content-Disposition: attachment
```

If `preview_path` is null, fetch by workspace path:
`GET /api/v1/files?path=Projects/.../episode.mp3`.

Follow-ups (`POST` same conversation id) keep context — "make it shorter", "more energy".

**Local source material** (a document, script, or deck on this machine): upload
it first, then reference the returned workspace path in the task message:

```
POST https://www.getonepress.com/api/v1/files?name=report.pdf&dir=Uploads
Authorization: Bearer $ONEPRESS_API_KEY
Content-Type: application/octet-stream

<raw file bytes>          # `dir` optional, default "Uploads"

→ 201 {"path":"Uploads/report.pdf","name":"report.pdf","size":1234}
```

Then e.g. `{"message": "Make a podcast episode from Uploads/report.pdf …"}`.
Max 50MB. Personal workspace only.

MCP alternative: server `https://www.getonepress.com/api/mcp`, tools
`onepress_create_task` / `onepress_task_status` / `onepress_list_tasks` /
`onepress_upload_file` / `onepress_download_artifact` (base64 for file bytes).

## Report back

- `answer` — the agent's summary
- The **local path** where you saved the MP3 (fetched via the artifact endpoint);
  it also lives in the user's OnePress workspace at
  https://www.getonepress.com/app (preview/share there)
- Conversation id/title

## Errors

401 bad key · 402 out of credits · 409 busy (keep polling) · 429 rate limited.

## Notes

- Be honest: audio is produced on getonepress.com, not here. Never fabricate files or links.
- Never ask the user to paste their API key into chat — env/config only.
