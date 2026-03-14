# Recording & Meeting Detection Flow

## 1. SDK Initialization

On app startup (`main.js:327`), `initSDK()` calls `RecallAiSdk.init()` with the API URL. This starts the SDK's background process that monitors the system for meeting windows (Zoom, Google Meet, Teams, Slack).

## 2. Meeting Detection

The Desktop SDK fires a `meeting-detected` event (`main.js:342`) when it detects a meeting app window.

**Event payload:**
- `evt.window.id` — unique window identifier
- `evt.window.platform` — `'zoom'`, `'google-meet'`, `'teams'`, `'slack'`
- `evt.window.title` — may be empty initially
- `evt.window.url` — may be empty initially

**What happens:**
1. Store the event in the `detectedMeeting` global
2. Show a native macOS notification with the platform name
3. Send `meeting-detection-status` IPC to the renderer (UI badge update)

## 3. Meeting Updated

A `meeting-updated` event (`main.js:390`) fires later as metadata (title, URL) becomes available. It updates `detectedMeeting` and retroactively patches the note title if one was already created.

## 4. Joining a Detected Meeting

Two entry points, both leading to `joinDetectedMeeting()` (`main.js:1757`):

| Trigger | Path |
|---------|------|
| Notification click | Calls `joinDetectedMeeting()` directly (`main.js:373`) |
| UI button | Renderer calls IPC → `ipcMain.handle('joinDetectedMeeting')` (`main.js:1752`) |

**`joinDetectedMeeting()` logic:**
1. Bring the main window to front
2. Wait 800ms for the window to be ready
3. Call `createMeetingNoteAndRecord(platformName)`

## 5. Create Note & Start Recording

`createMeetingNoteAndRecord()` (`main.js:1087`):

1. Create a meeting note object (ID, title, transcript array, platform)
2. Save it to `meetings.json` via `fileOperationManager`
3. Register in `activeRecordings` and `global.activeMeetingIds`
4. Send `open-meeting-note` IPC to the renderer to navigate to the note
5. Get an upload token from Recall.ai API via `createDesktopSdkUpload()`
6. Call `RecallAiSdk.startRecording({ windowId, uploadToken })` to begin capturing audio

## 6. During Recording

While recording is active, the SDK fires `realtime-event` events (`main.js:609`):

| Event Type | Handler |
|------------|---------|
| `transcript.data` | `processTranscriptData()` — appends to meeting transcript |
| `transcript.provider_data` | `processTranscriptProviderData()` |
| `participant_events.join` | `processParticipantJoin()` |
| `video_separate_png.data` | `processVideoFrame()` |

State changes are tracked via `sdk-state-change` events (`main.js:564`), updating `activeRecordings` and notifying the renderer via `recording-state-change` IPC.

## 7. Recording Ended

The `recording-ended` event (`main.js:485`) triggers the following sequence:

### 7a. Update the note
`updateNoteWithRecordingInfo()` (`main.js:1639`):
1. Find the meeting note in `meetings.json` by `recordingId`
2. Replace `"Recording: In Progress..."` with `"Recording: Completed at <timestamp>"`
3. Set `recordingComplete: true` and `recordingEndTime`
4. Save to disk

### 7b. Auto-generate AI summary
If `meeting.transcript.length > 0`:
1. Replace note content with `"Generating summary..."` placeholder
2. Send `summary-update` IPC to the renderer
3. Call `generateMeetingSummary()` with a streaming callback — sends `summary-update` IPC as chunks arrive for real-time editor updates
4. Save final summary, set `hasSummary: true`

### 7c. Upload the recording
After a 3-second delay:
1. Get a fresh upload token via `createDesktopSdkUpload()`
2. Call `RecallAiSdk.uploadRecording({ windowId, uploadToken })`
3. Falls back to upload without token if token fetch fails

### 7d. Notify the renderer
Send `recording-completed` IPC with the meeting ID.

## 8. Manual Recording (Alternative Path)

Triggered from the note editor UI without a detected meeting (`main.js:~860`):
1. `RecallAiSdk.prepareDesktopAudioRecording()` → returns a key
2. Create upload token, register in `activeRecordings`
3. `RecallAiSdk.startRecording({ windowId: key, uploadToken })`
4. Stopped via `stopManualRecording` IPC → `RecallAiSdk.stopRecording()`
5. The `recording-ended` event handles the rest (same as above)

## Visual Summary

```
SDK monitors system windows
        │
        ▼
  meeting-detected ──► notification + UI badge
        │
        ▼ (later)
  meeting-updated ──► patch title/URL
        │
  User clicks notification or UI button
        │
        ▼
  joinDetectedMeeting()
        │
        ▼
  createMeetingNoteAndRecord()
        ├── Create note in meetings.json
        ├── Open note in UI
        ├── Get upload token
        └── RecallAiSdk.startRecording()
                │
                ▼
        realtime-event (transcript, participants, video)
                │
                ▼
        recording-ended
                │
                ▼
        updateNoteWithRecordingInfo()
                ├── Mark "Recording: Completed"
                ├── Generate AI summary (streaming)
                └── Save to meetings.json
                │
                ▼ (3s delay)
        uploadRecording() → Recall.ai API
                │
                ▼
        recording-completed IPC → renderer
```
