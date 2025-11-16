# Spec Requirements: Voice Dictation for Notes

## Initial Description
voice dictation for notes.  plugin doc loc: /Users/benjaminmackenzie/Dev/flutter_dictation/docs/DICTATION_PLUGIN_INSTRUCTIONS.md

## Requirements Discussion

### First Round Questions

**Q1:** I assume the dictation entry point lives directly inside the Basic Notes composer (tap mic, dictate, auto-save transcript to the selected VIP contact). Is that correct, or do you expect a standalone “Dictate Note” flow before choosing the contact?
**Answer:** Dictation lives inside the notes composer—you tap the mic from that screen and the resulting text lands in the same raw notes field for the chosen contact.

**Q2:** I’m thinking we only persist the transcribed text (no audio file) and store it in the same notes table/structure as typed notes, with a flag indicating it came from dictation. Should we also save raw audio or intermediate metadata from the Flutter plugin?
**Answer:** Only the transcribed text should be persisted. It lives in the existing notes field, and adding a flag indicating the note came from dictation is desirable. No audio or intermediate metadata needs to be stored.

**Q3:** For the Flutter plugin (per `/Users/benjaminmackenzie/Dev/flutter_dictation/docs/DICTATION_PLUGIN_INSTRUCTIONS.md`), I’m assuming we’ll use on-device streaming transcription and show real-time text with basic controls (start, pause, cancel, submit). Do you also want phrase-level undo/voice commands (e.g., “delete last sentence”) in V1?
**Answer:** We expect to use on-device streaming transcription, but today we don’t show real-time text or expose start/pause/cancel controls—those patterns still need to be defined. No additional voice commands were requested.

**Q4:** To keep the scope manageable, I’m assuming V1 is iOS-only (per the plugin) and requires network connectivity for note sync but should keep the transcript locally if offline until sync resumes. Should offline dictation be considered out of scope for now?
**Answer:** Ideally offline dictation is in scope, but it’s unclear whether the existing plugin already handles that. Guidance is needed on whether plugin updates are required to support offline capture and later sync.

**Q5:** For privacy and permissions, I’m assuming we’ll request microphone access the first time users tap the mic icon and show a brief rationale modal describing why dictation is needed. Do you want any additional consent copy or telemetry requirements?
**Answer:** Current privacy/permission flows are sufficient; no extra consent copy or telemetry requirements are needed.

**Q6:** Are there any edge cases or exclusions you already know you want to defer (e.g., multi-language dictation, profanity filtering, speaker diarization, or editing transcripts after saving)?
**Answer:** No specific exclusions—languages and profanity filtering aren’t priorities, and speaker diarization or post-save editing workflows weren’t previously considered. Audio storage isn’t planned.

### Existing Code to Reference
No similar existing features identified for reference.

**Similar Features Identified:**
- Feature: Dictation Flutter plugin docs - Path: `/Users/benjaminmackenzie/Dev/flutter_dictation/README.md`
- Components to potentially reuse: Refer to plugin controls and waveform patterns described in the README.
- Backend logic to reference: None specified beyond existing notes persistence.

### Follow-up Questions

**Follow-up 1:** For dictation controls: since we currently don’t show real-time text or expose start/pause/cancel, should V1 introduce those controls along with live transcript rendering, or do you prefer a single “Hold to dictate” style where transcription appears only after release?
**Answer:** The desired UI already exists conceptually: tap mic to start recording/transcribing, show a waveform with an X on the left (cancel) and a checkmark on the right (stop and insert the transcript into the text box). That is the target interaction pattern.

**Follow-up 2:** Offline handling (Q4): do we treat offline dictation as out of scope, or should the plugin capture transcripts locally and queue them until the app can sync with Supabase?
**Answer:** Offline dictation should ideally be supported, though it’s unclear if the plugin can already manage it; we may need to extend the plugin if it doesn’t.

**Follow-up 3:** Privacy/permissions (Q5): beyond the standard iOS microphone prompt, do you want additional consent copy, a settings toggle for analytics/telemetry, or any in-app explanations about how dictation data is used?
**Answer:** No additional privacy or telemetry flows are required beyond the default microphone permission prompt.

**Follow-up 4:** Scope boundaries (Q6): are there specific languages, profanity filtering, speaker diarization, editing workflows, or audio storage features you definitely want to exclude from this spec?
**Answer:** No—languages and profanity filters are not concerns, the team isn’t familiar with speaker diarization, and no special editing or audio storage requirements are defined.

**Follow-up 5:** Existing code reuse: can you point me to current notes composer UI/state management files (or other components) we should reference so we stay consistent with the typed-notes experience?
**Answer:** Instead of UI code references, the main reusable asset is the dictation/note-taking plugin itself; see `/Users/benjaminmackenzie/Dev/flutter_dictation/README.md` for implementation details.

**Follow-up 6:** Visual assets: if you have any mockups or screenshots for the mic entry point or dictation flow, could you drop them into `agent-os/specs/2025-11-16-voice-dictation-for-notes/planning/visuals/`?
**Answer:** No screenshots are available; reference the plugin README for behavioral guidance.

## Visual Assets
No visual assets provided.

## Requirements Summary

### Functional Requirements
- Provide a mic entry point directly within the notes composer; tapping it initiates dictation for the active VIP contact.
- Use the Flutter dictation plugin for on-device streaming transcription; show a waveform UI with cancel (X) and confirm (checkmark) actions flanking it.
- When confirm is tapped, insert the transcript into the existing notes text area and mark the note as dictation-sourced via a flag.
- Persist only the transcribed text in the existing notes storage; no audio files or intermediate metadata need to be saved.
- Support offline dictation if feasible, buffering transcripts locally until they can sync to Supabase.

### Reusability Opportunities
- Reference `/Users/benjaminmackenzie/Dev/flutter_dictation/README.md` and associated plugin docs for control behaviors and waveform presentation.
- Reuse the existing notes data model and composer UI where possible, adding a dictation-sourced flag instead of creating new storage.

### Scope Boundaries
**In Scope:**
- Integrating dictation controls (mic button, waveform, cancel/check actions) inside the notes composer.
- On-device transcription via existing Flutter plugin, including insertion into notes and dictation flagging.
- Evaluating/adding offline buffering if the plugin supports it or can be extended.

**Out of Scope:**
- Audio file storage, speaker diarization, profanity filtering, or multilanguage-specific handling for this phase.
- Advanced voice commands (undo/delete last sentence) and post-save editing workflows beyond what the notes composer already allows.

### Technical Considerations
- Flutter (iOS-first) implementation using existing dictation plugin located at `/Users/benjaminmackenzie/Dev/flutter_dictation/`.
- Need to confirm whether the plugin already supports offline capture; may require plugin updates to queue transcripts until Supabase sync is available.
- Data model impact limited to adding a dictation flag on notes and ensuring synced storage works for both typed and dictated entries.
