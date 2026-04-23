# YouTube → Recall.it Pipeline (thought experiment)

Date: 2026-04-23

## The problem

I watch a lot of YouTube videos that I want to feed into [recall.it](https://recall.it)
for later review / knowledge extraction. I don't want to context-switch while I'm
watching — stopping to copy URLs or tag things kills the flow. I also don't want
to lose the videos in my YouTube history where they become effectively
unsearchable.

Recall.it doesn't (yet) have a public API for adding items programmatically, so
the final hop has to stay manual for now.

## The pipeline

```
watch video ──► add to YT playlist ──► script extracts URLs ──► paste into recall.it
   (manual)        (manual, 1 click)        (automated)            (manual, for now)
```

The key idea: **the only live friction is one click to "Save to playlist"**.
Everything else is either automated or batched.

### Stage 1 — Manual playlist building

- Create a dedicated YouTube playlist (e.g. `to-recall`) — unlisted.
- While watching, hit "Save" → playlist. That's the whole active step.
- Optionally a second playlist `recalled` to move items into after I've processed
  them, so `to-recall` acts like an inbox.

### Stage 2 — Automated URL extraction

Options, roughly in order of preference:

1. **YouTube Data API v3** (`playlistItems.list`).
   - Need an API key + OAuth for a private/unlisted playlist owned by me.
   - Returns `videoId`, `title`, `publishedAt`, channel, etc.
   - Cleanest, most stable.
2. **`yt-dlp --flat-playlist --print url`**.
   - Zero auth for unlisted playlists if I have the link.
   - Good fallback / offline option.
3. **Takeout / CSV export**.
   - Works but batch-only and slow feedback loop. Skip.

Output shape I want:

```
https://youtu.be/<id>  <tab>  <title>  <tab>  <channel>
```

Plain TSV so I can pipe it into anything later.

Run it as a local script (`scripts/pull-playlist.ts` or similar) invoked by hand
or on a cron / launchd timer. No server needed.

### Stage 3 — Manual paste into recall.it

- Script writes the URL list to clipboard and/or a dated file in `inbox/`.
- I paste into recall.it in one batch — weekly, say.
- When recall.it ships an API, replace this step with an HTTP POST loop and the
  whole pipeline becomes fully automated from click-to-save onward.

## The Simon Willison angle

Separately — Simon W talked about something in a video I want to revisit
(**TODO: find the video, probably via recall.it search once these are ingested**).
The gist I remember: treating your own notes / reading list as a searchable,
LLM-queryable corpus rather than a pile. That's exactly what recall.it is doing,
and it's the reason this pipeline is worth building — the value isn't the list
of URLs, it's being able to later ask "what have I watched about X?" and get an
answer grounded in things I actually chose to save.

So the real thought experiment is: **playlist = capture, recall.it = index**.
Keep capture zero-friction; let the index do the heavy lifting asynchronously.

## Open questions

- De-duping: if I re-add a video, I don't want to re-ingest it. Track already-
  processed IDs in a local file (`.processed-ids`).
- Do I want timestamps / my own notes per video? Probably yes eventually —
  YouTube's "Save with note" doesn't exist, so the natural place is a small
  companion file keyed by video ID.
- Transcripts: once a video is in recall.it, do I also want the transcript
  pulled (via `yt-dlp --write-auto-sub`) and stored alongside? Would make
  search dramatically better but is a v2 concern.

## Next steps

1. Create the `to-recall` YouTube playlist.
2. Write `scripts/pull-playlist.ts` against the Data API.
3. Run it manually for a week, see what the volume actually looks like before
   automating further.
4. Revisit when recall.it publishes an API.
