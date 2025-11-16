# Specification: Voice Dictation for Notes

## Goal
Enable VIP users to dictate notes directly inside the existing notes composer using the in-house Flutter dictation plugin, capturing low-latency transcripts (including offline when available) that flow into the same storage and workflows as typed notes.

## User Stories
- As a relationship-focused user, I want to tap a mic icon inside the notes composer so that I can capture thoughts hands-free without leaving the contact context.
- As a busy professional, I want clear cancel/confirm controls with live waveform feedback so that I trust when my dictation starts, stops, or is discarded.
- As a mobile-first user, I want dictated text to save even if I lose connectivity so that my notes stay consistent when the app comes back online.

## Specific Requirements

**Composer Entry Point & Controls**
- Add a mic icon adjacent to the existing text field actions in the notes composer; hide/disable it on non-iOS platforms until parity exists.
- Tapping the mic transitions the composer into a dictation state that displays a centered waveform strip with an `X` icon on the left (cancel) and a checkmark on the right (stop/save).
- Maintain visual consistency with Basic Notes patterns (typography, spacing, button sizes) and ensure controls remain reachable with the on-screen keyboard visible.
- Provide an elapsed-time badge or subtle timer while listening to reassure users the session is active.

**Dictation Session Lifecycle**
- Initialize `NativeDictationService` with `WaveformController` when the composer widget loads; dispose them when the composer unmounts.
- `startListening` fires on mic tap, subscribes to `status`, `result`, and `audioLevel` events, and transitions UI states (`ready → listening → stopped/cancelled`).
- `stopListening` finalizes transcription, resets waveform buffer, and emits a final transcript to the text field; `cancelListening` aborts and clears interim text.
- Guard against concurrent sessions by disabling the mic while `isListening == true` and auto-timeout after a configurable max duration (e.g., 5 minutes).

**Transcription Handling & Note Persistence**
- Append final transcripts to the active note text controller, preserving caret position and allowing manual edits before submission.
- Mark each saved note with a `source = dictation` flag (or boolean column) so downstream systems can differentiate dictated vs typed entries.
- Persist only the textual transcript; do not upload audio buffers, waveform samples, or partial metadata.
- Ensure autosave / manual save flows used for typed notes also fire for dictated notes with no schema divergence.

**Offline Readiness & Sync**
- Detect plugin `supportsOnDeviceRecognition` status; allow dictation to proceed offline when Apple’s on-device packs exist, otherwise surface a lightweight warning about needing connectivity.
- Buffer dictated transcripts locally (same draft model as typed notes) until Supabase sync succeeds; show a queued state in the UI if sync is pending.
- If the plugin indicates offline packs are missing, offer a help link directing users to iOS Settings → General → Keyboard → Dictation Languages to download packs.

**Plugin Integration & Platform Channels**
- Reuse `/Users/benjaminmackenzie/Dev/flutter_dictation` native path: invoke `initialize`, `startListening`, `stopListening`, `cancelListening`, and subscribe to `audioLevel` for waveform rendering.
- Mirror plugin error codes/messages into user-friendly toasts/snackbars and log them for diagnostics.
- Keep platform channel calls on user actions to satisfy iOS main-thread permission requirements.
- Reset `WaveformController` after stop/cancel events to avoid stale visual data.

**Error & Permission UX**
- Trigger microphone + speech permission prompts the first time the mic is tapped and handle denial states by showing inline guidance to enable permissions in Settings.
- Surface distinct messages for permission denied, recognizer errors, and plugin initialization failures; allow retry without forcing a full screen reload.
- Provide haptic/audio feedback on session start/stop (subject to platform conventions) to reinforce state changes for accessibility.

## Visual Design
No visual assets were provided; mirror the layout cues from the existing notes composer and the control arrangements described in `/Users/benjaminmackenzie/Dev/flutter_dictation/README.md` (waveform centered, cancel icon on left, confirm on right).

## Existing Code to Leverage

**`/Users/benjaminmackenzie/Dev/flutter_dictation` plugin**
- Supplies `NativeDictationService`, waveform widgets, and the Swift managers needed for low-latency dictation.
- Includes `AudioControlsDecorator` that can wrap the notes composer to gain waveform + cancel/confirm UI scaffolding with minimal custom code.
- Provides on-device recognition support when Apple’s offline packs are available, fulfilling the offline requirement without custom DSP work.
- Documents platform channel contracts and error handling conventions to keep app-side logic consistent.

## Out of Scope
- Persisting or uploading audio recordings, waveform samples, or intermediate recognition metadata.
- Advanced voice-edit commands (e.g., “delete last sentence,” “insert comma”) beyond what the text field already supports.
- Dedicated dictation-only screens outside the existing notes composer flow.
- Android dictation implementation (remains future work until plugin parity exists).
- Speaker diarization, profanity filtering, or multilingual UX beyond whatever Apple provides by default.
- Post-save transcript editing workflows beyond the standard notes editor.
