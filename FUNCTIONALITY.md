# Muesli — Recall.ai Desktop App Functionality

## Overview

Muesli is an Electron desktop app that uses the [Recall.ai Desktop SDK](https://www.recall.ai/) to automatically detect, record, and transcribe meetings. It generates AI-powered summaries and provides a markdown note editor per meeting. An embedded Express server proxies Recall.ai API calls to keep credentials server-side.

---

## Core Features

### 1. Automatic Meeting Detection

Muesli watches for active meetings on the following platforms:

- Zoom
- Google Meet
- Slack
- Microsoft Teams

When a meeting is detected, the user is prompted to join and record it. The SDK handles permission requests from the OS automatically.

### 2. Recording

**Online meetings (auto-detected)**
- The SDK joins the detected meeting and captures video, audio, screen shares, and participant events.
- Recording is triggered via the "Record Meeting" button on the home screen.

**In-person meetings (manual)**
- The "Record In-person Meeting" button starts a desktop audio capture session.
- Uses `RecallAiSdk.prepareDesktopAudioRecording()` to capture system audio without joining a video call.

### 3. Real-Time Transcription

- Transcripts are generated using **AssemblyAI v3 streaming** via the Recall.ai API.
- Transcript chunks are pushed to the UI in real time via IPC events as the recording progresses.
- The full transcript is stored with the meeting and displayed in the note editor.

### 4. AI-Powered Meeting Summaries

- Uses **OpenAI** (`gpt-5-mini`) to generate summaries from the meeting transcript.
- Two modes:
  - **Full** — returns the complete summary at once.
  - **Streaming** — delivers the summary in chunks for real-time display.
- Summaries can be triggered manually from the UI or automatically on recording completion.
- Requires `OPENAI_API_KEY` in `.env`.

### 5. Meeting Management

- All meetings are persisted locally in `meetings.json` (in the Electron app data directory).
- The home screen lists meetings grouped by date, each showing title, timestamp, participant count, and a transcript preview.
- Meetings can be deleted from the list.

**Meeting data shape:**
```json
{
  "id": "string",
  "title": "string",
  "platform": "zoom | meet | slack | teams",
  "date": "timestamp",
  "participants": [],
  "transcript": "string",
  "summary": "string",
  "notes": "string",
  "recordingStatus": "idle | recording | stopped"
}
```

### 6. Note Editor

Each meeting has a dedicated note editor page with:

- **Editable title** — click to rename the meeting.
- **Markdown editor** — powered by SimpleMDE with a full formatting toolbar (bold, italic, headings, lists, links, etc.).
- **Auto-save** — notes are saved automatically with debouncing to avoid excessive disk writes.
- **Metadata display** — shows date, time, and participant list.
- **Live transcript/events panel** — displays incoming transcript chunks and SDK events during an active recording.

The sidebar also contains UI placeholders for planned actions:
- Copy link / Copy text / Share via Email or Slack
- AI actions: list action items, write follow-up email, list Q&A

---

## Architecture

```
Muesli
├── Main Process          src/main.js
│   ├── RecallAI SDK init & event handling
│   ├── IPC handler registration
│   ├── OpenAI summary generation
│   └── File I/O (meetings.json)
│
├── Express Proxy Server  src/server.js  (port 13373)
│   └── GET /start-recording — creates Recall.ai upload token
│
├── Preload Script        src/preload.js
│   └── Context bridge → exposes electronAPI & sdkLoggerBridge
│
└── Renderer Processes
    ├── Home page         src/index.html + src/renderer.js
    └── Note editor       src/pages/note-editor/
```

---

## IPC API (Renderer ↔ Main)

### Invokable handlers

| Handler | Description |
|---|---|
| `loadMeetingsData()` | Load all meetings from disk |
| `saveMeetingsData(data)` | Persist meetings to disk |
| `deleteMeeting(meetingId)` | Remove a meeting |
| `generateMeetingSummary(meetingId)` | Generate AI summary (full) |
| `generateMeetingSummaryStreaming(meetingId)` | Generate AI summary (streaming chunks) |
| `startManualRecording(meetingId)` | Begin desktop audio recording |
| `stopManualRecording(recordingId)` | Stop active recording |
| `checkForDetectedMeeting()` | Check whether a meeting has been auto-detected |
| `joinDetectedMeeting()` | Join and start recording a detected meeting |
| `getActiveRecordingId(noteId)` | Get the active recording ID for a given note |
| `debugGetHandlers()` | List all registered IPC handlers (debug) |

### Events pushed from main → renderer

| Event | Description |
|---|---|
| `recording-state-change` | Recording state changed (`idle`, `recording`, `stopped`) |
| `recording-completed` | Recording finished |
| `transcript-updated` | New transcript chunk received |
| `summary-generated` | AI summary complete |
| `summary-update` | Streaming summary chunk |
| `participants-updated` | Participant list changed |
| `video-frame` | Video frame data |
| `open-meeting-note` | Navigate to a meeting note |
| `meeting-detection-status` | Meeting detection status update |
| `meeting-title-updated` | Meeting title changed |

---

## Express Proxy Server

**Port:** `13373`

| Endpoint | Method | Description |
|---|---|---|
| `/start-recording` | `GET` | Creates a Recall.ai upload token. Configures real-time event streams for participants, video frames, and transcripts. Returns `{ status, upload_token }`. |

This server exists to keep `RECALLAI_API_KEY` out of the renderer process. All Recall.ai API calls are routed through it.

---

## RecallAI SDK Events

| Event | Description |
|---|---|
| `meeting-detected` | Stores the meeting in `detectedMeeting`, shows an OS notification (clicking it calls `joinDetectedMeeting()`), sends `meeting-detection-status: true` to renderer |
| `meeting-updated` | Updates `detectedMeeting` with the latest title/URL; if a note already exists for this meeting, retroactively updates its title in `meetings.json` and sends `meeting-title-updated` to renderer |
| `meeting-closed` | Clears `detectedMeeting` and `global.activeMeetingIds` entry, sends `meeting-detection-status: false` to renderer |
| `recording-started` | No handler — event is not subscribed to in the app |
| `recording-ended` | Calls `updateNoteWithRecordingInfo()`, then after a 3 s delay fetches a new upload token and calls `RecallAiSdk.uploadRecording()` |
| `permissions-granted` | No-op — logs to console only |
| `upload-progress` | Logs progress to console; when `progress === 100` logs upload completion (no UI update) |
| `sdk-state-change` | Updates `activeRecordings` state (`recording` → adds entry, `paused` → updates state, `idle` → removes entry); sends `recording-state-change` to renderer |
| `realtime-event` | Dispatches to sub-handlers by event type (see below) |
| `error` | Shows an OS notification with the error type and message. The SDK auto-restarts on unexpected shutdown. |

### Not implemented

The following SDK events are available but not handled by the app:

| Event | What it provides |
|---|---|
| `permission-status` | Per-permission status on startup (`accessibility`, `screen-capture`, `microphone`) — each can be `granted`, `not_requested`, or `denied`. Useful for showing setup guidance to the user. |
| `media-capture-status` | Fired when window capture is interrupted or resumed (e.g. meeting window is on an inactive virtual desktop). Provides `type` (`video`/`audio`) and `capturing` boolean. |
| `participant-capture-status` | Fired when a participant's media capture is interrupted or resumed (e.g. their video tile is no longer visible due to crowding). Provides `type` (`video`/`audio`/`screenshare`), `participantId`, and `capturing` boolean. |
| `network-status` | Fired when network connectivity changes — `disconnected` (down for 5 s) or `reconnected` (back up for 5 s). Important: if disconnected for 2 minutes, all recordings are stopped automatically. |
| `shutdown` | Fired when the SDK shuts down normally. Provides `code` and `signal`. |

### Real-time event sub-types

The single `realtime-event` listener dispatches to four sub-handlers based on `evt.event`:

| `evt.event` | Handler | What it does |
|---|---|---|
| `transcript.data` | `processTranscriptData()` | Extracts words and speaker info, appends a formatted `"Speaker: text"` line to the meeting's transcript array in `meetings.json`, and sends `transcript-updated` to the renderer |
| `transcript.provider_data` | `processTranscriptProviderData()` | Reads the raw AssemblyAI speaker diarization payload to track the current unknown speaker index — used as a fallback when participant name is unavailable |
| `participant_events.join` | `processParticipantJoin()` | Adds or updates the participant (name, id, isHost, platform, joinTime) in the meeting's `participants` array in `meetings.json`, and sends `participants-updated` to the renderer |
| `video_separate_png.data` | `processVideoFrame()` | Extracts the base64 PNG frame, participant info, frame type (`webcam` or `screenshare`), and timestamp, then sends a `video-frame` IPC event to the renderer — intentionally excluded from SDK logs to avoid flooding |

---

## Environment Variables

| Variable | Required | Description |
|---|---|---|
| `RECALLAI_API_URL` | Yes | Recall.ai regional base URL (e.g. `https://us-east-1.recall.ai`) |
| `RECALLAI_API_KEY` | Yes | Recall.ai API key |
| `OPENAI_API_KEY` | No | OpenAI key — required for AI meeting summaries |

---

## Key Dependencies

| Package | Purpose |
|---|---|
| `@recallai/desktop-sdk` | Meeting detection, recording, transcription |
| `openai` | AI meeting summaries via GPT |
| `express` | HTTP proxy server |
| `react` | UI rendering |
| `easymde` / `simplemde` | Markdown note editor |
| `marked` | Markdown → HTML rendering |
| `axios` | HTTP client |
| `electron-forge` | App packaging and distribution |

---

## Build & Distribution

- Target platforms: **macOS** (DMG), **Linux** (Deb, RPM), **Windows** (Squirrel, Zip)
- macOS signing uses a custom fork of `@electron/osx-sign`
- `@recallai/desktop-sdk` is unpacked from the asar archive (native module requirement)
- No linting is configured (`pnpm run lint` is a no-op)

---

## SDK Deep Dive

### Data transmission: streaming vs. bulk upload

Recording data is handled in two distinct layers:

- **Real-time events are streamed continuously** during recording. The `/start-recording` endpoint configures a `desktop_sdk_callback` realtime endpoint that pushes transcript chunks, video frames, and participant events to the main process via `RecallAiSdk.addEventListener('realtime-event', ...)` as they happen.
- **The recording file itself is uploaded after the meeting ends** in a single post-hoc call. When `recording-ended` fires, the app waits 3 seconds then calls `RecallAiSdk.uploadRecording({ windowId, uploadToken })`.

The UX feels real-time (transcript appears live), but the underlying audio/video file is stored locally by the SDK until the meeting stops, then uploaded in one shot.

---

### Recording modes: what `windowId` actually means

The two recording modes differ in what the `windowId` represents, which determines what gets captured:

| | Record Meeting | Record In-Person |
|---|---|---|
| `windowId` source | `detectedMeeting.window.id` from the SDK's meeting detection | Key returned by `prepareDesktopAudioRecording()` |
| What it represents | A specific meeting app window (Zoom, Chrome tab with Meet, etc.) | The system's audio streams — all desktop audio |
| What is captured | Only that meeting window's audio/video | All desktop audio |
| Extra SDK call required | None | `prepareDesktopAudioRecording()` |
| `startRecording` / `stopRecording` / `uploadRecording` | Identical API | Identical API |
| Transcript | Yes (AssemblyAI streaming) | Yes (same pipeline) |

After the initial setup diverges, both modes are identical — the same `startRecording → recording-ended → uploadRecording` flow applies to both.

---

### Local storage

| What | Where |
|---|---|
| Meeting metadata, transcripts, notes, summaries | `~/Library/Application Support/muesli/meetings.json` (macOS) |
| Raw recording files (audio/video) | Managed internally by `@recallai/desktop-sdk` until `uploadRecording()` is called |

`meetings.json` grows indefinitely — entries are only removed when the user explicitly deletes a meeting. There is no TTL, size limit, or automatic cleanup.

---

### Known gaps: shutdown & recovery

#### Graceful quit (Cmd+Q)

There are **no `before-quit` or `will-quit` handlers** in the app. If the user quits while a recording is active:

- The in-memory `activeRecordings` state is lost
- `stopRecording()` is never called
- `uploadRecording()` is never called
- The meeting in `meetings.json` is left with `recordingStatus: 'recording'` permanently

The correct fix is a `before-quit` handler that calls `event.preventDefault()`, stops all active recordings, waits for `recording-ended`, completes the upload, then calls `app.quit()`.

> **Note:** Force-quit (`kill -9`, Activity Monitor → Force Quit) sends `SIGKILL`, which cannot be caught by any handler. There is no solution for that case.

#### Laptop lid / system sleep

There are **no `powerMonitor` handlers** for system suspend/resume. When the lid closes:

- The OS suspends the process — no quit events fire, no JS runs
- Any in-progress network connections (upload) are dropped
- When the lid opens, the process resumes mid-state; the SDK's internal capture state is uncertain
- The recording is likely corrupted or incomplete

The fix requires `powerMonitor.on('suspend')` (stop & save) and `powerMonitor.on('resume')` (attempt upload).

#### Startup recovery

On `app.whenReady()`, the only startup logic is creating `meetings.json` if it doesn't exist. There is no recovery:

- No scan for meetings stuck in `recordingStatus: 'recording'`
- No attempt to resume or mark crashed recordings as failed
- `activeRecordings` starts empty every launch

Meetings interrupted by a crash or force-quit will appear in the UI permanently stuck in a "recording" state with no transcript, no summary, and no path to recover the audio data.
