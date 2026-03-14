# @recallai/desktop-sdk

Version: 2.0.8 — Native SDK for recording desktop meetings. Supports macOS and Windows.

## What it does

Spawns a platform-specific native subprocess and communicates over stdio using a JSON-RPC-like protocol. The JS layer wraps that subprocess with async commands and an event emitter.

> **macOS:** binary is `desktop_sdk_macos_exe`
> **Windows:** binary name differs — check the installed package for the platform-specific executable.

Capabilities:
- Automatically detects active meetings (Zoom, Google Meet, Slack, Microsoft Teams)
- Captures video, audio, screen shares, and participant events
- Streams real-time transcripts via AssemblyAI v3
- Uploads recordings to the Recall.ai backend after the meeting ends

---

## Installation

```bash
npm install @recallai/desktop-sdk
# or
pnpm add @recallai/desktop-sdk
```

### Electron / asar requirement *(macOS and Windows)*

The native binary must be **unpacked from asar**. In `forge.config.js`:

```js
packagerConfig: {
  asar: { unpackDir: '**/node_modules/@recallai/**' }
}
```

The SDK handles path remapping automatically: it tries the `.asar.unpacked` path first, then falls back to the `.asar` path.

---

## Quick start

```js
import RecallAiSdk, {
  init, shutdown,
  startRecording, stopRecording,
  pauseRecording, resumeRecording,
  requestPermission, addEventListener,
  prepareDesktopAudioRecording,
} from '@recallai/desktop-sdk';

// 1. Initialize (starts the native subprocess)
await init({
  api_url: 'https://us-east-1.recall.ai',
  acquirePermissionsOnStartup: ['accessibility', 'screen-capture', 'microphone'],
  restartOnError: true,
});

// 2. Listen for a meeting to appear
addEventListener('meeting-detected', ({ window }) => {
  console.log('Meeting detected:', window);
});

// 3. Start recording
await startRecording({ windowId: window.id, uploadToken: '<token>' });

// 4. Stop recording
await stopRecording({ windowId: window.id });

// 5. Gracefully shut down
await shutdown();
```

---

## API

| Function | Signature | Notes |
|---|---|---|
| `init` | `(config: RecallAiSdkConfig) => Promise<null>` | Must be called first. Starts the native subprocess. |
| `shutdown` | `() => Promise<null>` | Gracefully stops the subprocess. |
| `startRecording` | `({ windowId, uploadToken }) => Promise<null>` | Begins recording a window. |
| `stopRecording` | `({ windowId }) => Promise<null>` | Stops recording. |
| `pauseRecording` | `({ windowId }) => Promise<null>` | Pauses an active recording. |
| `resumeRecording` | `({ windowId }) => Promise<null>` | Resumes a paused recording. |
| `requestPermission` | `(permission) => Promise<null>` | Requests a macOS permission. |
| `addEventListener` | `(type, callback) => void` | Subscribes to SDK events (see below). |
| `prepareDesktopAudioRecording` | `() => Promise<string>` | Prepares audio capture, returns a `windowId` token. |

---

## Config (`RecallAiSdkConfig`)

```ts
{
  api_url?: string;                       // or apiUrl — Recall.ai regional base URL
  acquirePermissionsOnStartup?: Permission[];
  restartOnError?: boolean;               // auto-restart subprocess on crash (up to 10 times)
  dev?: boolean;
  testMode?: boolean;
}
```

---

## Events

Subscribe with `addEventListener(type, callback)`.

| Event type | Payload | Notes |
|---|---|---|
| `'meeting-detected'` | `{ window }` | A supported meeting app window appeared. |
| `'meeting-updated'` | `{ window }` | Meeting window metadata changed. |
| `'meeting-closed'` | `{ window }` | Meeting window closed. |
| `'recording-started'` | `{ window }` | Recording begun. |
| `'recording-ended'` | `{ window }` | Recording stopped. |
| `'sdk-state-change'` | `{ sdk: { state: { code } } }` | State: `'idle'`, `'recording'`, `'paused'`. |
| `'media-capture-status'` | `{ window, type, capturing }` | `type`: `'video'` or `'audio'`. Fires when window capture is interrupted or resumed (e.g. meeting on inactive virtual desktop). |
| `'participant-capture-status'` | `{ window, type, capturing }` | `type`: `'video'`, `'audio'`, `'screenshare'`. Fires when a participant's media capture is interrupted (e.g. video tile no longer visible). |
| `'permissions-granted'` | `{}` | All requested permissions granted. |
| `'permission-status'` | `{ permission, status }` | Per-permission status on startup. `status`: `'granted'`, `'not_requested'`, or `'denied'`. Useful for showing setup guidance. |
| `'realtime-event'` | `{ window, event, data }` | SDK-level realtime events (transcript, video frames, participant joins — see below). |
| `'error'` | `{ window?, type, message }` | SDK error. |
| `'shutdown'` | `{ code, signal }` | Subprocess exited. Only useful for post-mortem cleanup — SDK is already dead at this point, cannot call `stopRecording` or `uploadRecording`. |
| `'network-status'` | `{ status }` | `'reconnected'` or `'disconnected'`. **Important:** if disconnected for 2 minutes, all recordings are stopped automatically. |
| `'log'` | `{ level, message, subsystem, category, window_id }` | Internal SDK log. |

### Real-time event sub-types

The `realtime-event` listener receives `evt.event` to distinguish sub-types:

| `evt.event` | What it provides |
|---|---|
| `transcript.data` | Words and speaker info from AssemblyAI streaming transcription. |
| `transcript.provider_data` | Raw AssemblyAI speaker diarization payload; use to track unknown speaker indices as fallback. |
| `participant_events.join` | Participant joined: `name`, `id`, `isHost`, `platform`, `joinTime`. |
| `video_separate_png.data` | Base64 PNG frame, participant info, `type` (`webcam` or `screenshare`), timestamp. Note: can be very high volume — exclude from logs. |

---

## Permissions *(macOS only)*

`Permission` type accepts:

`'accessibility'` | `'screen-capture'` | `'microphone'` | `'system-audio'` | `'full-disk-access'`

These map to macOS system permission prompts. On Windows, the SDK does not use this permission model — `requestPermission` and `acquirePermissionsOnStartup` are macOS-only.

Request at startup via `acquirePermissionsOnStartup` in config, or on-demand via `requestPermission(permission)`.

Listen to `'permission-status'` events to check per-permission state and show setup guidance to the user.

---

## Recording modes

The SDK supports two recording modes. After setup, both use the identical `startRecording → recording-ended → uploadRecording` flow.

| | Online meeting | In-person (desktop audio) |
|---|---|---|
| `windowId` source | `window.id` from `meeting-detected` event | Token returned by `prepareDesktopAudioRecording()` |
| What is captured | That meeting window's audio/video only | All desktop audio |
| Extra SDK call required | None | `prepareDesktopAudioRecording()` |
| Transcript | Yes (AssemblyAI streaming) | Yes (same pipeline) |

---

## Data flow: streaming vs. upload

Recording data is handled in two distinct layers:

- **Real-time events are streamed continuously** during recording. Configure a `desktop_sdk_callback` realtime endpoint when creating an upload token — the SDK pushes transcript chunks, video frames, and participant events to your handler via `'realtime-event'` as they happen.
- **The recording file itself is uploaded after the meeting ends.** When `'recording-ended'` fires, call `uploadRecording({ windowId, uploadToken })` to send the buffered audio/video to the Recall.ai backend. The SDK holds the file locally until this call is made.

---

## Local storage

| Data | Location | Notes |
|---|---|---|
| SDK diagnostic logs (macOS) | `~/Library/Application Support/<EXE_NAME>/recall_diagnostic.jsonl` + `recall_diagnostic_delta.jsonl` | Written by the native binary. `EXE_NAME` = Electron bundle ID at runtime. |
| SDK diagnostic logs (Windows) | `%APPDATA%\<EXE_NAME>\recall_diagnostic.jsonl` + `recall_diagnostic_delta.jsonl` | Same files, Windows path convention. |
| GStreamer debug dots | `os.tmpdir()` in production; `/tmp/gst.nocommit/` (macOS/Linux) or equivalent temp dir (Windows) in dev mode | Controlled by `RECALLAI_DESKTOP_SDK_DEV` env var. |
| Recordings | None — streamed to Recall.ai backend | Since v2.0.0, captured data is buffered by the SDK internally then uploaded via `uploadRecording()`. Nothing is written to a user-visible path. |

---

## Subprocess internals

- Commands are sent as JSON lines to the process's stdin; responses arrive on stdout.
- Pending commands are tracked by UUID; the SDK resolves/rejects the corresponding Promise on receipt.
- Auto-restarts on unexpected crash (up to 10 times) when `restartOnError: true`.

---

## Shutdown & recovery

### Graceful quit

Call `shutdown()` before the app exits. In an Electron app, use a `before-quit` handler:

```js
app.on('before-quit', async (event) => {
  event.preventDefault();
  // Stop all active recordings, wait for 'recording-ended', upload, then:
  await shutdown();
  app.quit();
});
```

Without this, any active recordings will be silently lost — `stopRecording` and `uploadRecording` are never called, and the SDK subprocess exits with captured data still buffered locally.

> **Note:** Force-quit (`kill -9`, Activity Monitor → Force Quit) sends `SIGKILL` which cannot be caught. There is no solution for that case.

### The `shutdown` event and signals

The `shutdown` event fires when the subprocess exits, with `{ code, signal }`. It is **not suitable for completing an upload** — by the time it fires, the SDK is already dead. Its practical use is narrower:

- **`before-quit`** — right place for graceful shutdown; SDK is still alive.
- **`shutdown`** — only useful for post-mortem cleanup (e.g. marking interrupted recordings as failed in your data store).

Known signals from observed behavior: `SIGINT` (normal quit), `SIGTERM` (killed by OS/process manager). `SIGKILL` likely never triggers the event. Treat signal values as informational until confirmed by the SDK docs.

### System sleep

Register `powerMonitor` handlers to handle laptop lid close:

```js
import { powerMonitor } from 'electron';

powerMonitor.on('suspend', () => {
  // Stop active recordings and save state
});

powerMonitor.on('resume', () => {
  // Attempt upload of any stopped recordings
});
```

Without these, recordings in progress when the lid closes will likely be corrupted or incomplete — the OS suspends the process mid-capture with no quit events fired.

### Startup recovery

On startup, scan your data store for meetings stuck in an active recording state (e.g. `recordingStatus: 'recording'`). These represent recordings interrupted by a crash or force-quit. Mark them as failed or attempt recovery — the SDK starts with no memory of previous sessions.
