# Spec Requirements: Basic Notes System

## Initial Description
Basic Notes System — Implement typed note-taking functionality where users can log interaction history and relationship context, with notes stored individually and associated with specific VIP contacts.

## Requirements Discussion

### First Round Questions

**Q1:** I’m assuming typed notes will be created from each VIP contact’s detail screen, with a lightweight composer that pre-selects that contact. Is that correct, or should we also support a global “quick note” entry that later assigns contacts?  
**Answer:** Support a global QuickNote entry that can later assign contacts. QuickNote screen layout: contact search bar at top, horizontal row of pinned favorites (scrollable if needed), then text entry box paired with a microphone button tied to the existing dictation plugin. Users can type or dictate, and dictated text populates the same input.

**Q2:** I’m thinking each note stores the author, contact, timestamp, full body text, and optionally quick-typed metadata (e.g., interaction type, sentiment, follow-up flag). Should we capture any additional structured fields such as tags or reminder dates at creation time?  
**Answer:** Keep it to raw text only plus any metadata we can capture automatically. No tags, reminder dates, or other user-entered structured fields for this release.

**Q3:** To keep relationship history trustworthy, I assume users can edit or delete their own notes with version history logged for auditing. Do we need full edit history, soft deletion, or is a simple update-in-place sufficient for v1?  
**Answer:** Simple edits in place are fine; we don’t need version history or soft delete for now.

**Q4:** For note browsing, I’m imagining a chronological feed per contact with infinite scroll and a condensed preview (first line + timestamp). Should we also add global search or filtering (by date range, contact, or keyword) in this initial release?  
**Answer:** The per-contact feed idea works, but we’ll display a “cleaned” version of each note—run through a grammar/clarity module with no information loss. In the feed we show the cleaned text with easy access to the raw note. Global search would be nice but isn’t critical for v1.

**Q5:** I assume every note ties to exactly one VIP contact to keep evidence references clear. Do we need to support linking a single note to multiple contacts or to external entities (e.g., companies) yet?  
**Answer:** Every note links to one contact only. No support for multiple contacts or other entities.

**Q6:** Since the roadmap mentions future voice dictation and signal extraction, I’m treating this feature as typed-only but storing data in a way those future systems can read (e.g., per-note UUIDs, structured metadata slots). Are there other downstream integrations we should account for now (message generation triggers, reminders, analytics)?  
**Answer:** Voice dictation is actually part of this release (using the existing Flutter plugin). Signal extraction and other downstream systems will come later, so just ensure the data model can support them in the future.

**Q7:** Are there any elements you explicitly want to exclude from this iteration (attachments, collaborative editing, import/export, etc.)?  
**Answer:** No additional exclusions were specified; focus on typed + voice capture, cleaned display, and per-contact feeds.

### Existing Code to Reference
No similar existing features identified for reference.

### Follow-up Questions

**Follow-up 1:** You described the QuickNote screen with both typing and a microphone that feeds in text from the dictation plugin. Should we treat voice dictation capture as an in-scope requirement for this Basic Notes System, or is the microphone UI just a placeholder until we ship the dedicated “Voice Dictation for Notes” roadmap item?  
**Answer:** Voice dictation capture is fully in scope now; it will use the existing Flutter plugin. The user can provide details if we need to inspect it later.

**Follow-up 2:** For the “cleaned note” module that rewrites notes into grammatically clear sentences, do you want us to persist that cleaned version alongside the raw note (e.g., separate columns/fields) or simply derive it on demand? Also, how should users access the raw text—tap-to-expand the note card, a “view raw note” action, something else?  
**Answer:** Persist the cleaned version alongside the raw note (not computed on demand). Users can tap to expand a note card, which reveals the raw note below the cleaned text.

## Visual Assets

### Files Provided:
No visual assets provided.

## Requirements Summary

### Functional Requirements
- Capture notes via typing or in-scope voice dictation (existing Flutter plugin) within the QuickNote screen and contact detail screens.
- Provide contact selection via search bar, pinned favorites row, and post-entry contact assignment.
- Store raw note text plus a persisted cleaned version, along with automatic metadata (author, contact, timestamps).
- Support edit-in-place for notes, with each note associated to exactly one contact.
- Render a per-contact chronological feed with infinite scroll showing cleaned notes and tap-to-expand raw text, with optional (non-critical) global search consideration.

### Reusability Opportunities
- Reuse the existing Flutter voice dictation plugin for microphone input.
- No additional existing UI or backend components were identified for reuse.

### Scope Boundaries
**In Scope:**
- Typed and voice-based note entry via QuickNote UI and contact detail screens.
- Persisted cleaned note text alongside raw text.
- Per-contact feed with infinite scroll and tap-to-expand raw view.
- Simple edit-in-place operations.

**Out of Scope:**
- User-entered tags, reminder dates, attachments, multi-contact linking, or collaborative editing.
- Version history, soft deletes, and advanced search/filtering (beyond potential future enhancements).
- Signal extraction and other downstream automation (handled in later roadmap items).

### Technical Considerations
- Datastore must hold both raw and cleaned note bodies, ready for future signal extraction and message generation.
- Integrate existing Flutter dictation plugin for voice capture; ensure QuickNote UI hooks into it cleanly.
- Maintain clear contact associations (one-to-one) to support evidence-based relationship tracking.

