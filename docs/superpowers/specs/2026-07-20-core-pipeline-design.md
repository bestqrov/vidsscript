# vidsscript — Core Pipeline Design

## Context

vidsscript is a SaaS web app: a user types a topic, the platform generates a
full narration script, splits it into scenes, synthesizes voice-over per
scene, generates a visual (image or video clip) per scene, then assembles
everything into one downloadable video (5–10 minutes long).

The whole project decomposes into four independent sub-projects:

1. **Core Pipeline** (this spec) — topic → script → scenes → audio → visuals
   → assembled video, via pluggable AI provider adapters.
2. **SaaS layer** — accounts, subscription packs, daily usage quotas,
   payment (provider TBD).
3. **Frontend** — topic input, progress view, download, plan/usage display.
4. **Mobile** (future) — Android/iOS clients reusing the same API-first
   backend. No mobile-specific work happens now; the backend must simply
   not assume a browser-only client.

This document covers sub-project 1 only.

## Constraints

- Deploy target: Coolify on a Hostinger VPS, 8 GB RAM.
- No AI provider API keys exist yet. All providers must be swappable via
  config with zero code changes.
- Generated videos are **never stored permanently**. They live in temp
  storage with a 48-hour expiry, after which both the DB record and the
  physical files are deleted. The UI must show the user a clear expiry
  warning next to the download link.
- Repo: `github.com/bestqrov/vidsscript`, deployed via Coolify.

## Stack

- **Node.js + Express** — orchestration is I/O-bound (parallel calls to
  external script/TTS/image/video APIs); Node's single-process event loop
  keeps memory overhead low on the 8 GB VPS compared to a multi-process
  worker model (e.g. Celery), leaving more RAM for FFmpeg.
- **MongoDB** (Atlas, connection string already provisioned) — job/script
  data is naturally nested/variable-shaped (scene count and metadata differ
  per job), a good fit for documents over relational rows.
- **BullMQ + Redis** — job queue for the multi-stage async pipeline, with
  built-in retry/backoff per stage.
- **FFmpeg** (via `fluent-ffmpeg`) — final assembly: Ken Burns effect on
  static images, scene concatenation, transitions, audio muxing.

## Data model

### Collection: `videojobs`

```
{
  _id,
  userId: ObjectId | null,       // null = anonymous/demo use
  topic: string,
  status: "pending" | "script" | "audio" | "visuals" | "assembling"
        | "done" | "failed",
  progress: number,              // 0-100
  script: {
    fullText: string,
    scenes: [{ index: number, text: string }]
  },
  scenes: [{
    index: number,
    text: string,
    audio: { path: string, durationSec: number, status: string },
    visual: { type: "image" | "video", path: string, prompt: string, status: string }
  }],
  output: { path: string, durationSec: number, sizeBytes: number } | null,
  error: string | null,
  createdAt: Date,
  expiresAt: Date                // TTL index; cleanup job also deletes files
}
```

A minimal `users` collection is stubbed now (`_id`, `email`) so `userId` has
somewhere to point once the SaaS layer lands; no auth logic is built in
this sub-project.

## Pipeline workflow (BullMQ queues)

1. `POST /api/videos { topic }` → create `VideoJob` (`status: pending`) →
   enqueue `script` job.
2. **Worker: script** — `ScriptProvider.generate(topic)` → full script +
   scene breakdown → save → `status: audio` → enqueue one `audio` job per
   scene.
3. **Worker: audio** — `TTSProvider.synthesize(scene.text)` per scene, in
   parallel → save audio file + duration → once all scenes have audio →
   `status: visuals` → enqueue one `visual` job per scene.
4. **Worker: visual** — `VisualProvider.generate(prompt)` per scene (prompt
   derived from scene text) → returns an image or a video clip depending on
   which adapter is configured → once all scenes have a visual →
   `status: assembling` → enqueue `assemble` job.
5. **Worker: assemble** — FFmpeg combines each scene (image + Ken Burns +
   audio, or video + audio), concatenates scenes with simple transitions,
   exports the final file → `status: done`, `output` set,
   `expiresAt = now + 48h`.
6. **Cleanup worker** (scheduled) — deletes physical files for jobs past
   `expiresAt`; Mongo's TTL index removes the DB document independently.

`GET /api/videos/:id` — polled by the frontend for `status` + `progress`.
On `status: done`, the response includes the download URL and `expiresAt`
so the UI can render the "download now, deleted in 48h" warning.

### Error handling

Each BullMQ job retries with backoff on failure (e.g. upstream API
timeout/5xx). After retries are exhausted, the job sets `status: failed`
and `error`, and any partial files for that job are cleaned up immediately
rather than waiting for the 48h expiry.

## Provider adapters

One interface per capability; concrete implementations selected via env
config, so adding/swapping a real API key later requires no code changes:

```
ScriptProvider.generate(topic)      -> { fullText, scenes: [{index, text}] }
TTSProvider.synthesize(text)        -> { path, durationSec }
VisualProvider.generate(prompt)     -> { type: "image"|"video", path }
```

Until real API keys are supplied, each interface gets a mock/stub
implementation so the full pipeline is testable end-to-end without any
external dependency.

## Out of scope (this sub-project)

- Authentication, subscription packs, usage quotas, billing (sub-project 2).
- Any rendered frontend UI beyond what's needed to manually exercise the
  API during development (sub-project 3).
- Mobile clients (sub-project 4) — only constraint honored here is keeping
  the backend API-first.
