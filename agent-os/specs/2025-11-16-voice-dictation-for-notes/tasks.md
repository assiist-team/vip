# Task Breakdown: Voice Dictation for Notes

## Overview
- Total Tasks: 24
- Inputs: `spec.md`, `planning/requirements.md`, `/Users/benjaminmackenzie/Dev/flutter_dictation/README.md`
- Standards to honor: frontend (accessibility/components/css/responsive), global (coding-style/commenting/conventions/error-handling/performance/security/validation/tech-stack), testing (`standards/testing/test-writing.md`), backend models/queries for notes persistence
- Key dependencies: Basic Notes System data model, Flutter dictation plugin native path, Supabase auth/session context, Apple on-device dictation packs for offline support

## Task List

### 1. Data Model & Persistence Updates
**Dependencies:** Existing notes schema from Basic Notes System

- [ ] 1.0 Extend notes storage contract
  - [ ] 1.1 Add `source` enum/boolean column (e.g., `is_dictated` or `note_source` enum) with migration + RLS updates ensuring owners-only edits
  - [ ] 1.2 Update Supabase PostgREST policies/tests to cover dictated notes CRUD
  - [ ] 1.3 Expose new field in API typings/DTOs consumed by Flutter data layer

**Acceptance Criteria:**
- Migration applies cleanly via Supabase CLI with rollback plan documented
- RLS prevents cross-user access and passes regression tests
- Flutter data models compile with new source flag and default `typed` state

### 2. Flutter Service & State Wiring
**Dependencies:** Task Group 1 (field availability)

- [ ] 2.0 Integrate `NativeDictationService`
  - [ ] 2.1 Create Riverpod provider managing service lifecycle (`initialize`, `dispose`, `startListening`, etc.) with retry/backoff per plugin docs
  - [ ] 2.2 Wire `WaveformController` and event subscriptions (`status`, `audioLevel`, `result`) into shared composer state
  - [ ] 2.3 Add repository/update flows to set `note.source = dictation` when transcripts save
  - [ ] 2.4 Unit-test provider logic (mock plugin stream) covering start/stop/cancel/error transitions

**Acceptance Criteria:**
- Provider exposes deterministic state for UI (ready/listening/stopped)
- Tests cover at least: successful dictation, cancel, permission error, plugin failure
- Memory leaks avoided (disposing subscriptions on widget teardown)

### 3. Composer UI & Controls
**Dependencies:** Task Group 2

- [ ] 3.0 Embed mic controls within notes composer
  - [ ] 3.1 Add mic icon entry point in composer toolbar (hide on non-iOS)
  - [ ] 3.2 Implement waveform strip with X (cancel) and checkmark (confirm) positioned per spec; ensure accessible labels + hit targets
  - [ ] 3.3 Display elapsed time badge + listening state copy; add haptic/audio cues on start/stop when supported
  - [ ] 3.4 Append final transcript to text controller while preserving cursor; allow manual edits after dictation
  - [ ] 3.5 Add widget tests covering mic toggle, cancel, confirm, mixed typing+dictation, accessibility labels

**Acceptance Criteria:**
- UI matches spec layout and Material/Cupertino theming
- Screen-reader focus order includes mic, cancel, confirm, and text field updates
- Tests pass across golden/device form factors (portrait/landscape)

### 4. Offline/Resilience Handling
**Dependencies:** Task Groups 2-3

- [ ] 4.0 Support offline dictation + sync
  - [ ] 4.1 Detect plugin `supportsOnDeviceRecognition`; show inline tip when offline pack missing with link to Settings instructions
  - [ ] 4.2 Leverage existing draft storage to queue dictated transcripts until Supabase sync succeeds; surface “Queued” badge/state
  - [ ] 4.3 Implement background retry + exponential backoff for syncing queued dictated notes
  - [ ] 4.4 Add tests (unit/integration) simulating offline capture, app relaunch, and eventual consistency after reconnect

**Acceptance Criteria:**
- Dictation succeeds with airplane mode when offline packs installed; queued notes sync automatically later
- Users receive non-blocking warning when offline packs absent or OS falls back to server recognition
- Telemetry captures offline queue length + failures

### 5. Error Handling, Permissions & QA
**Dependencies:** Task Groups 2-4

- [ ] 5.0 Harden permissions + failure UX
  - [ ] 5.1 Ensure Info.plist contains microphone + speech usage strings matching spec copy
  - [ ] 5.2 Build permission-denied state inside composer with CTA to open Settings
  - [ ] 5.3 Display distinct toasts/dialogs for plugin initialization failures, timeouts, or transcription errors and log to Sentry
  - [ ] 5.4 Author automated tests (unit/widget) covering permission denied, cancel, plugin error surfacing, and dictation-to-save flow
  - [ ] 5.5 Execute manual QA checklist: first-run permission, offline dictation, long session timeout, rapid cancel/confirm taps, sync conflict resolution; log results under `implementation/`

**Acceptance Criteria:**
- Permission prompts only fire once per platform rules; denial path documented with fallback UI
- Errors never leave composer stuck in listening state; recovery works without restart
- QA log + automated tests stored alongside feature artifacts; CI remains green

