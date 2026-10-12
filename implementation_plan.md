# Google Drive & Google Sheets Voice & Chat Hub

An integrated voice recording and chat application connected directly to Google Drive and Google Sheets. Users can record voice messages and send chat messages; audio files are uploaded directly to Google Drive, shareable links are generated, and all entries are logged in real time to a connected Google Sheet.

## User Review & Critical Decisions

> [!IMPORTANT]
> The clarifying questions were skipped, so the recommended architecture has been selected to deliver the most complete, powerful experience:

- **Confirmed Architecture**: Hybrid AI Chat Assistant & Voice Memo Logger. The app functions both as an intelligent conversational partner (with speech-to-text transcription powered by Gemini) and as a direct logging terminal that archives audio files into Google Drive and message records into Google Sheets.
- **Google Workspace OAuth Scopes**:
  - `https://www.googleapis.com/auth/drive.file` (Permits creating and managing app files and folders in Google Drive with least-privilege security)
  - `https://www.googleapis.com/auth/spreadsheets` (Permits creating and updating Google Sheets logs)
- **Automatic Setup & Organization**: The app will automatically create and organize a dedicated `"Voice & Chat App"` folder in Google Drive and a `"Voice & Chat Logs"` Google Sheet, while also offering a selector if the user wants to point to an existing sheet.
- **Language Support**: Full support for Urdu and English in speech transcription, chat messages, and interface localization.

---

## 1. Overview & Core Concept

- **What It Does**: A real-time voice and text messaging hub where recorded voice messages are converted to audio blobs, uploaded to Google Drive with shareable URLs, transcribed via Gemini API, and automatically appended into a synchronized Google Sheet alongside conversation history.
- **Target Audience / Persona**: Users, creators, and professionals needing seamless voice logging, team memos, or AI conversational records stored securely in their own Google Workspace cloud.
- **Key Value**: Eliminates manual file uploading and spreadsheet entry. Record a voice memo or send a message, and within seconds the audio file is safely preserved in Drive and cataloged with a direct hyperlink and transcript in Google Sheets.

---

## 2. User Experience & Visual Design

### Key User Flows

1. **Authentication Flow**:
   - User signs in securely with Google using official Google Sign-In button.
   - App checks for existing `"Voice & Chat App"` folder in Google Drive and `"Voice & Chat Logs"` Google Sheet; auto-initializes them if not present.
2. **Voice Recording & Upload Flow**:
   - User taps or holds the Microphone button with real-time waveform visualization, timer, and cancel/preview options.
   - Upon completion, audio is rendered, uploaded as a `.webm` / `.mp3` file to Google Drive, and made viewable.
   - Audio is concurrently transcribed into text via Gemini.
   - The Drive shareable link, transcript, timestamp, file size, and duration are appended as a new row in Google Sheets.
3. **Chat & AI Assistant Flow**:
   - User types or speaks messages. In AI Assistant mode, Gemini generates contextual responses.
   - User and Assistant messages are reflected in the chat timeline and archived to Google Sheets.
4. **Live Workspace Explorer Flow**:
   - An integrated panel displays the live synced Google Sheet rows (with click-to-play audio links) and Google Drive file history.
   - One-click shortcuts to open the live Google Sheet or Google Drive folder in a new tab.

### Visual Identity & Theme

- **Aesthetic Direction**: Clean, contemporary productivity dashboard inspired by Google Workspace design standards and modern audio tools.
- **Color Palette**:
  - Neutral Canvas: Crisp white (`#FFFFFF`) / Off-white (`#F8FAFC`) with subtle cool slate borders (`#E2E8F0`).
  - Brand Accents: Google Drive Emerald Green (`#0F9D58`) and Google Sheets Mint (`#0F9D58`), paired with Google Blue (`#1A73E8`) for primary actions.
  - Audio & Recording Accents: Crimson (`#EA4335`) during live recording and pulse states.
- **Typography**:
  - Primary Sans: `Plus Jakarta Sans` for titles, buttons, and navigation.
  - Tabular Numerals: `JetBrains Mono` / `tabular-nums` for timers, audio durations, timestamps, and spreadsheet cell coordinates.
- **Layout Rhythm**:
  - Left/Center: Dual-mode Chat and Voice Recording studio with interactive waveform, quick record bar, and message stream.
  - Right: Synced Google Sheets inspector and Google Drive file repository viewer.

---

## 3. Key Product Decisions & Trade-Offs

- **Client-Side Google Workspace API Integration**:
  - *Chosen Approach*: Use client-side Firebase Auth with Google OAuth provider tokens passed in `Authorization: Bearer <token>` headers directly to Google Drive v3 and Google Sheets v4 REST APIs.
  - *Why*: Conforms strictly to AI Studio iframe and dynamic environment requirements, eliminating OAuth redirect URI mismatch errors.
- **Direct Drive Resumable/Multipart Audio Upload**:
  - *Chosen Approach*: Upload recorded audio blobs directly to `https://www.googleapis.com/upload/drive/v3/files?uploadType=multipart` with `drive.file` scope.
  - *Why*: Instant, high-speed upload without server bottlenecks, with immediate return of Drive File ID and `webViewLink`.
- **Structured Spreadsheet Schema**:
  - *Columns*: `[ID, Date & Time, Message Type, Sender, Transcript / Message, Drive Audio Link, Duration, Status]`.
  - *Formula Hyperlinks*: Injects `=HYPERLINK("drive_url", "Open Audio")` formulas so users can open recordings directly within Google Sheets.

---

## 4. Technical Architecture & Data Strategy

```
┌────────────────────────────────────────────────────────────────────────┐
│                        Google Workspace Voice & Chat Hub               │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
          ┌─────────────────────────┼─────────────────────────┐
          ▼                         ▼                         ▼
┌──────────────────┐      ┌──────────────────┐      ┌──────────────────┐
│  Audio Recorder  │      │   Chat Engine    │      │ Workspace Sync   │
│  & Visualizer    │      │  (Text + AI)     │      │ Manager          │
└─────────┬────────┘      └─────────┬────────┘      └─────────┬────────┘
          │                         │                         │
          │ Recorded Blob           │ Prompt / Speech-to-Text │ Token & IDs
          ▼                         ▼                         ▼
┌────────────────────────────────────────────────────────────────────────┐
│                         Service Integration Layer                      │
│                                                                        │
│   ┌────────────────────┐   ┌─────────────────┐   ┌─────────────────┐   │
│   │  Google Drive v3   │   │ Google Sheets v4│   │  Gemini 2.5 API │   │
│   │  - Create Folder   │   │ - Auto-Create   │   │  - Transcription│   │
│   │  - Upload Audio    │   │ - Append Rows   │   │  - AI Chat      │   │
│   │  - Web View Links  │   │ - Read Grid     │   │                 │   │
│   └────────────────────┘   └─────────────────┘   └─────────────────┘   │
└────────────────────────────────────────────────────────────────────────┘
```

### Data Structures

```typescript
interface VoiceChatRecord {
  id: string;
  timestamp: string;
  type: 'voice' | 'text';
  sender: 'user' | 'assistant';
  content: string; // text message or speech-to-text transcript
  driveFileId?: string;
  driveViewLink?: string;
  audioDuration?: string;
  audioBlobUrl?: string;
  syncStatus: 'local' | 'uploading_drive' | 'updating_sheet' | 'synced' | 'error';
}

interface WorkspaceConfig {
  driveFolderId: string | null;
  driveFolderName: string;
  sheetId: string | null;
  sheetName: string;
  sheetUrl: string | null;
}
```
