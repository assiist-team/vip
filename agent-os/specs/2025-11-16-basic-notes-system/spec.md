# Specification: Basic Notes System

## Goal
Deliver a dependable typed-and-dictated notes experience that lets VIP users capture relationship context anywhere, keeps each note tied to a single contact, and stores both raw and cleaned text so future intelligence features (signal extraction, message generation) can rely on trustworthy evidence.

## User Stories
- As a busy relationship manager, I want a QuickNote composer that’s always available so I can jot context the moment I remember it, then attach it to the right VIP contact later.
- As a user who prefers speaking, I want to tap a microphone and dictate notes using our existing plugin so my thoughts land in the same note pipeline without extra steps.
- As someone reviewing a contact’s history, I want a clean, easy-to-read feed but also be able to expand any entry to see the original wording so I can trust the nuance I captured.
- As a user editing mistakes, I want to update a note in place without worrying about breaking downstream relationship history.

## Dependencies & Downstream Contracts
- **Contact Sync & VIP List Management (`2025-11-16-contact-sync-vip-list-management`):** Requires reliable VIP contact data, pinned favorites, and contact search APIs. Notes cannot exist without a `contact_id` from that feature. Contact deletions must either cascade notes or enforce referential integrity (decision deferred to contacts spec, but this feature must expect `ON DELETE RESTRICT`).
- **Voice Dictation Plugin (existing Flutter plugin):** QuickNote and contact detail composers must integrate the current dictation plugin for iOS, streaming text into the same controller as typed input. Any platform constraints (iOS-first) follow the plugin’s limitations.
- **Future Signal Extraction / Message Generation specs:** Store raw text, cleaned text, timestamps, author, and source metadata so downstream jobs can compute signals without reprocessing user edits. Each note must expose a stable UUID for referencing evidence in future specs.

## Specific Requirements

### QuickNote Capture Surface
- **Entry Points**
  - Provide a global QuickNote screen accessible from the primary nav (exact trigger defined by UX) plus a composer embedded on each contact detail screen.
  - QuickNote layout: contact search bar on top, horizontal pinned-favorites carousel beneath it (scrollable when overflow), then a dual-mode composer containing a multiline text field and microphone button.
- **Contact Selection**
  - Users can select a contact before or after drafting. Unassigned drafts must not be saved; require contact selection before final submission.
  - Pinned favorites originate from the contacts feature (recent VIPs or explicit favorites); tapping sets the contact immediately.
- **Composer Behavior**
  - Text field supports typing and receives dictation output seamlessly (same controller).
  - Show remaining character guidance only if limits exist (not defined now); otherwise allow long-form text with graceful scrolling inside the field.

### Voice Dictation Integration
- Microphone button invokes the existing Flutter dictation plugin; plugin output must stream into the text controller in real time and append where the cursor sits.
- Show live recording state (waveform or timer) consistent with accessibility guidelines; include cancel/send affordances.
- When dictation ends, pause the plugin and keep text editable before submission.
- Handle plugin errors (permissions denied, plugin unavailable) with inline messaging and fall back to typing.

### Note Cleaning & Storage
- Every saved note generates two bodies:
  - `raw_body`: exact user text (typed or dictated).
  - `cleaned_body`: persisted output from a grammar/clarity module that fixes casing/punctuation without changing meaning.
- Cleaning happens at save-time; no on-demand recomputation. Store `cleaned_body` alongside metadata to avoid mismatches when editing later.
- Data model (Supabase Postgres) MUST include: `id uuid primary key`, `user_id uuid references auth.users`, `contact_id uuid references contacts`, `raw_body text not null`, `cleaned_body text not null`, `source enum('typed','dictated')`, `created_at timestamptz default now()`, `updated_at timestamptz`, `cleaned_at timestamptz`, optional `pinned_contact_rank` snapshot for analytics, and future-proof JSONB `auto_metadata` for system-detected hints (kept empty for now).
- Enforce RLS so users only read/write their own notes (`auth.uid() = notes.user_id`).

### Editing & Lifecycle
- Allow authors to edit their own notes in place from either the feed or contact detail. Editing rewrites both `raw_body` and `cleaned_body` (cleaner reruns after editing).
- Provide delete ability that permanently removes the row (no soft delete requirement yet). Confirm before deleting and ensure UI refreshes the feed.
- Maintain optimistic updates on the client with Supabase mutation hooks; on failure revert to the previous state.

### Contact-Centric Feed & Browsing
- Each contact detail screen shows a reverse-chronological feed of cleaned notes with infinite scroll/pagination (e.g., page size 20 with lazy loading).
- Card contents:
  - Cleaned text preview (up to ~3 lines) plus timestamp, author avatar/initials, and source badge (Typed / Dictated).
  - Tap/click expands the card vertically to reveal the raw text underneath the cleaned body, maintaining scroll position.
- Provide a visual indicator if the note was edited (e.g., “Edited · 2d ago”).
- Global search/filtering is optional for v1; include architecture hooks (Supabase full-text index on `cleaned_body` and `raw_body`) but keep UI backlog unless time allows.

### Contact Binding Rules
- Every note requires a single `contact_id`. Multi-contact linking, company-level notes, or broadcast logging are explicitly out-of-scope.
- If a contact is removed from the VIP list, decide whether to retain their notes; default stance: contacts remain in database even if unsynced, so notes keep their pointer. If contact deletion is allowed in the future, this feature must surface meaningful errors when attempting to save a note for a missing contact.

### Quick Access & Draft Handling
- Provide local draft persistence (per user, per device) so unsaved text survives app backgrounding. Drafts should not sync to Supabase or be visible cross-device.
- When a draft is restored, prompt the user to finish assigning a contact before saving.

### Observability & Error Handling
- Log note creation/edit/delete failures to Sentry with note ID (if available), contact ID, and action type.
- Surface network errors inline underneath the composer; retry without duplicating notes.
- Respect validation standards: block empty submissions, trim whitespace, and keep copy aligned with mission tone.

## Visual Design
No visual assets were provided. Follow VIP’s typography, spacing, and accessibility standards (minimum 44px tap targets, sufficient color contrast, VoiceOver labels for microphone states).

## Existing Code to Leverage
- **Existing Flutter dictation plugin:** Reuse the current plugin APIs for microphone capture; no new native work should be required.
- **Contact module UI components:** Reuse the search field, contact chips, and avatar widgets from the Contact Sync/VIP spec to keep visual consistency.
- **Global standards (`agent-os/standards/global/*.md`):** Apply coding style, error handling, validation, and performance guidelines already ratified.

## Out of Scope
- User-entered metadata such as tags, manual sentiment, reminders, attachments, photos, or file uploads.
- Multi-contact notes, shared notes, collaborative editing, or comment threads.
- Version history, audit trails, or soft deletion (future enhancement).
- Automated signal extraction, reminder scheduling, or message-generation triggers (handled by later specs).
- Offline-first syncing, conflict resolution, or cross-device draft sync.

## Acceptance Checklist
- [ ] Global QuickNote composer with contact search, pinned favorites, text field, and microphone button operates end-to-end.
- [ ] Dictation plugin writes into the same controller as typing and gracefully handles permission/availability errors.
- [ ] Notes save both `raw_body` and `cleaned_body` fields and show cleaned text in feeds with tap-to-expand raw view.
- [ ] Users can edit or delete their own notes; data and UI stay in sync with Supabase mutations.
- [ ] Contact detail screen shows infinite-scrolling feed scoped to that contact, ordered newest-first, with edited indicators and source badges.
- [ ] RLS and validation prevent accessing or mutating notes belonging to other users.

