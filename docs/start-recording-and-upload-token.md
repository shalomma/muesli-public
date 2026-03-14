# `startRecording` Calls & Upload Token

## Why `RecallAiSdk.startRecording` Appears 4 Times

There are two recording flows, and the detected-meeting flow has fallback branches for token failures.

### 1. Manual desktop recording (`main.js:935`)

Inside the `startManualRecording` IPC handler — the happy path for recordings started from the note editor UI without a detected meeting.

```js
RecallAiSdk.startRecording({ windowId: key, uploadToken: uploadData.upload_token });
```

### 2–4. Detected-meeting recording (`createMeetingNoteAndRecord`, `main.js:1087`)

Three calls cover all token-fetching outcomes:

```
try {
  uploadData = await createDesktopSdkUpload();

  if (!uploadData || !uploadData.upload_token) {
    // ② Line 1210 — token fetch returned empty/null
    startRecording({ windowId })
  } else {
    // ③ Line 1222 — happy path, valid token
    startRecording({ windowId, uploadToken })
  }
} catch (error) {
  // ④ Line 1238 — token fetch threw an error
  startRecording({ windowId })
}
```

| # | Location | Scenario |
|---|----------|----------|
| 1 | `main.js:935` | Manual recording (happy path) |
| 2 | `main.js:1210` | Detected meeting — token fetch returned empty |
| 3 | `main.js:1222` | Detected meeting — happy path with token |
| 4 | `main.js:1238` | Detected meeting — token fetch threw error |

Calls 2 and 4 are **degraded fallbacks** that record without an upload token rather than failing entirely.

## What is `createDesktopSdkUpload`?

`createDesktopSdkUpload()` (`main.js:305`) fetches an upload token by calling the local Express server:

```
Electron main process
        │
        │  GET http://localhost:13373/start-recording
        ▼
Express server (server.js:11)
        │
        │  POST {RECALLAI_API_URL}/api/v1/sdk_upload/
        │  Headers: Authorization: Token {RECALLAI_API_KEY}
        │  Body: {
        │    recording_config: {
        │      transcript: { provider: { assembly_ai_v3_streaming: {} } },
        │      realtime_endpoints: [{
        │        type: "desktop_sdk_callback",
        │        events: [
        │          "participant_events.join",
        │          "video_separate_png.data",
        │          "transcript.data",
        │          "transcript.provider_data"
        │        ]
        │      }]
        │    }
        │  }
        ▼
Recall.ai API
        │
        │  Returns { upload_token: "..." }
        ▼
Express → Electron → RecallAiSdk.startRecording({ windowId, uploadToken })
```

The Express server exists as a proxy so the `RECALLAI_API_KEY` never touches the Electron renderer process.

## Who Issues the Token?

**Recall.ai's backend** (`/api/v1/sdk_upload/`) issues it. The token is opaque and authorizes the Desktop SDK to upload recorded audio to Recall.ai for processing.

## With Token vs Without Token

### With upload token (intended path)

```js
RecallAiSdk.startRecording({ windowId, uploadToken })
```

The token carries the recording configuration. This enables:
- **Upload** of recorded audio to Recall.ai
- **Transcription** via AssemblyAI (server-side)
- **Realtime events** streamed back to the SDK (transcript data, participant joins, video frames)
- Proper session association when `uploadRecording()` is called at `recording-ended`

### Without upload token (fallback)

```js
RecallAiSdk.startRecording({ windowId })
```

The SDK still **captures audio locally**, but without a pre-configured server-side session:
- No guaranteed transcription or realtime events
- The subsequent `uploadRecording()` call at `recording-ended` may not succeed
- Logged as an error: `'Failed to get upload token. Recording without upload token.'`

This is purely a degraded fallback — record something rather than fail entirely.
