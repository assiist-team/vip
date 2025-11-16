# Task Breakdown: Basic Notes System

## Overview
- Total Tasks: 36
- Inputs: `spec.md`, `planning/requirements.md`
- Standards to honor: backend (migrations/models/api/queries), frontend (components/css/accessibility/responsive), global (coding-style/commenting/conventions/error-handling/performance/security/validation/tech-stack), testing (`standards/testing/test-writing.md`)
- Dependencies: `2025-11-16-contact-sync-vip-list-management` for contact data, existing Flutter dictation plugin, Supabase auth/session guarantees from `2025-11-16-auth-login-feature`

## Task List

### 1. Data Model & Supabase Foundations
**Dependencies:** None

- [ ] 1.0 Shape Supabase schema + plumbing
  - [ ] 1.1 Author migration for `notes` table with columns + indexes per spec (raw/cleaned bodies, source enum, timestamps) following `standards/backend/migrations.md`
  - [ ] 1.2 Implement enum + helper domain (`note_source`) and ensure seed/default aligns with Flutter constants
  - [ ] 1.3 Configure RLS policies tying `notes.user_id` to `auth.uid()`; add row-level insert/update/delete checks
  - [ ] 1.4 Create SQL tests (3-4) covering RLS visibility, FK integrity, and enum constraints per `standards/testing/test-writing.md`
  - [ ] 1.5 Expose Supabase Edge function or RPC (if needed) for cleaned-body updates, or confirm direct PostgREST writes suffice; document decision in spec appendix

**Acceptance Criteria:**
- Migration applies cleanly via Supabase CLI
- RLS prevents cross-user access yet allows owners full CRUD
- Enum values match Flutter model (`typed`, `dictated`)
- Tests pass locally and in CI

### 2. Flutter Data Layer & Models
**Dependencies:** Task Group 1

- [ ] 2.0 Build Dart models + repositories
  - [ ] 2.1 Define `Note` Freezed model + JSON serialization for all columns, including optional edit timestamp and auto metadata
  - [ ] 2.2 Create Riverpod repository/service wrapping Supabase PostgREST (create, update, delete, paginate)
  - [ ] 2.3 Implement DTO ↔ domain mappers ensuring cleaned/raw text coexist
  - [ ] 2.4 Add repository unit tests (3-5) covering serialization, pagination parsing, and error propagation
  - [ ] 2.5 Wire logging + Sentry breadcrumbs inside repository per `standards/global/error-handling.md`

**Acceptance Criteria:**
- Model build_runner passes with no analyzer warnings
- Repository methods return typed results and bubble structured errors
- Tests cover happy path + Supabase failure handling

### 3. Note Cleaning Service
**Dependencies:** Task Group 2

- [ ] 3.0 Implement “clean note” pipeline
  - [ ] 3.1 Decide implementation approach (local heuristic vs. LLM/Edge Function); document in README/spec appendix
  - [ ] 3.2 Build service that accepts raw text, returns cleaned text, and timestamps `cleaned_at`
  - [ ] 3.3 Expose sync API to Flutter layer (Riverpod provider) so save/edit flows await cleaned result before final mutating call
  - [ ] 3.4 Handle failures by preserving raw text and surfacing retry UI/toast
  - [ ] 3.5 Write focused tests (2-3) validating no information loss (basic heuristics) and failure fallbacks

**Acceptance Criteria:**
- Cleaning service persists deterministic output for same input
- Save/edit flows never write cleaned text as empty unless raw empty (which is blocked)
- Errors captured in Sentry with context

### 4. QuickNote Composer & Dictation Integration
**Dependencies:** Task Groups 2-3

- [ ] 4.0 Build global QuickNote UI
  - [ ] 4.1 Implement screen scaffold: search bar, pinned favorites carousel, composer card (text + mic)
  - [ ] 4.2 Integrate existing Flutter dictation plugin; ensure permission prompts and live-state UI
  - [ ] 4.3 Sync dictation output with text controller (play nice with manual typing, cursor positions)
  - [ ] 4.4 Handle contact selection (search, pinned favorites) with validation gating save action
  - [ ] 4.5 Add draft persistence (local storage) keyed by user/device and clear upon successful save
  - [ ] 4.6 Build widget tests (4-5) covering typing, dictation toggle, contact assignment, draft restore, and validation copy

**Acceptance Criteria:**
- Composer supports both typing and dictation seamlessly
- Pinned favorites scroll horizontally with accessible semantics
- Unsaved drafts survive app backgrounding and are cleared after submission
- Tests cover primary interactions and pass in CI

### 5. Contact Detail Composer & Feed
**Dependencies:** Task Groups 2-4

- [ ] 5.0 Enhance contact detail screen
  - [ ] 5.1 Embed inline composer tied to selected contact (no contact search needed)
  - [ ] 5.2 Create paginated feed provider per contact (infinite scroll, reverse chronological)
  - [ ] 5.3 Design note card UI: cleaned preview, timestamp, source pill, edit indicator, avatar/initials
  - [ ] 5.4 Implement tap-to-expand raw text reveal with smooth animation and a11y labels
  - [ ] 5.5 Provide edit + delete affordances inline, with confirmation dialogs following `standards/global/validation.md`
  - [ ] 5.6 Add feed widget tests (3-4) verifying pagination, expand/collapse, edit state, and deletion

**Acceptance Criteria:**
- Feed stays synced via repository updates (optimistic updates resolved)
- Raw text accessible via tap while maintaining scroll position
- Dictated notes display source badges correctly
- Tests pass and confirm expansions work with screen readers

### 6. Optional Global Browse/Search (Scaffolding)
**Dependencies:** Task Groups 2-5

- [ ] 6.0 Lay groundwork for future global search while shipping MVP toggle
  - [ ] 6.1 Add Supabase index/full-text search migration but keep API behind feature flag
  - [ ] 6.2 Stub global search UI entry (e.g., placeholder button) that links to backlog explanation
  - [ ] 6.3 Document extension points for search/filter (README snippet, provider hooks)

**Acceptance Criteria:**
- Indexes exist without impacting current flows
- Feature flag prevents unfinished UI from appearing in production builds
- Documentation clearly outlines activation steps

### 7. Error Handling, Telemetry & QA
**Dependencies:** Task Groups 1-6

- [ ] 7.0 Harden experience + validate
  - [ ] 7.1 Instrument Sentry + analytics events (note_create, note_edit, note_delete, dictation_start/end, clean_failed)
  - [ ] 7.2 Add inline error states for Supabase failures, dictation plugin issues, and cleaning service timeouts
  - [ ] 7.3 Compile automated test inventory from previous groups (approx. 12-20 tests) and ensure CI target run
  - [ ] 7.4 Execute manual QA checklist: dictation permissions, airplane mode, long notes, rapid edits, contact deletion edge cases
  - [ ] 7.5 Document residual risks/future work (multi-contact notes, attachments, search UI) in spec appendix

**Acceptance Criteria:**
- All automated tests referenced above are green
- QA log stored under `implementation/` detailing scenarios + outcomes
- Telemetry events appear in dev environment dashboards or logcat
- Known gaps captured with owners/next steps

