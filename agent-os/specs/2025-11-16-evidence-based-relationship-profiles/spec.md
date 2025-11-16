# Specification: Evidence-Based Relationship Profiles

## Goal
Create trustworthy, contact-level relationship profiles that live on the existing contact detail screen, exposing FORD (Family, Occupation, Recreation, Dreams) facts plus a narrative summary backed by traceable note evidence. Profiles refresh automatically whenever new notes are saved, capture audit metadata, accept user edits, and fuel downstream message-generation pipelines without blocking on pending updates.

## User Stories
- As a user reviewing a VIP contact, I want to see up-to-date FORD facts and a concise relationship summary alongside the contact’s info so I immediately know what matters most.
- As a meticulous user, I want to tap into evidence for auto-generated facts (e.g., note excerpts) so I can confirm accuracy without leaving the screen.
- As a user correcting information, I want to override any FORD field or the summary manually so the profile matches what I know, even if I don’t link it to a note.
- As the message-generation service, I need a consistent, queryable profile snapshot so I can ground prompts in authoritative relationship data even while new updates are processing.

## Dependencies & Downstream Contracts
- **Notes System (future spec `2025-11-16-basic-notes-system`)**: Every note save must emit an event or trigger the AI workflow that recalculates the profile. Failure paths need retries so profile data doesn’t lag indefinitely.
- **Message Generation (future specs like `message-rules`, `context-retrieval`):** Profiles must be available via Supabase tables/views that downstream retrieval code can JOIN when assembling prompts. Message generation uses the latest *confirmed* snapshot; in-flight updates only replace the snapshot after validation.
- **Contact Sync & VIP List (`2025-11-16-contact-sync-vip-list-management`):** Profiles attach to the VIP contact records (`contacts.id`). RLS must ensure users only see their own contact profiles and evidence.
- **Telemetry & Auditing (Sentry / Supabase logging):** Store model + version metadata for each automated update so investigators can reconstruct how a fact was produced.

## Functional Overview
1. **Single Surface:** The contact detail screen gains a “Relationship Profile” section with:
   - Summary paragraph at the top.
   - FORD facts grouped with labels and optional evidence affordances.
2. **Automated Refresh:** Saving a note enqueues a workflow that:
   - Reads the note content + existing profile.
   - Produces candidate FORD updates and summary adjustments strictly from explicit statements.
   - Writes a new profile snapshot plus evidence references and AI metadata.
   - Triggers message-generation jobs, which still read the prior committed snapshot until the new one is published.
3. **Manual Edits:** Users can edit FORD entries or the summary. Manual entries are tagged `user_asserted=true` and may omit evidence. Automated updates never overwrite manual entries without user confirmation; instead they create suggestions (future enhancement) or skip fields currently marked user-asserted.
4. **Evidence Storage:** Every auto fact stores `note_id`, `note_excerpt`, `note_timestamp`, `model_name`, and `model_version`. UI keeps evidence subtle (e.g., “View note” chip) to avoid clutter.
5. **Versioning:** Maintain `profiles_snapshot` records with `published_at` so message generation can reference the most recent published snapshot atomically.

## Data Model
The data layer separates three responsibilities so each concern stays simple and auditable:
- `relationship_profiles` holds the single “envelope” per contact (summary text, manual-edit flags, pointer to the published snapshot). Think of it as the lightweight header record that is easy to join with contacts.
- `relationship_profile_facts` stores the actual FORD entries plus their evidence metadata. Multiple rows per category let us capture several facts (e.g., multiple Occupation details) while tracking whether each was auto-generated or user-asserted.
- `relationship_profile_snapshots` keeps immutable freeze frames that downstream message-generation jobs consume. Snapshots let the UI continue updating live data without exposing half-written facts to long-running generation flows.

Create/extend tables (Supabase PostgreSQL):

### `relationship_profiles`
Purpose: the authoritative 1:1 profile header tied to a contact; keeps the latest summary, manual-edit indicators, and which snapshot is published for downstream consumers.

| Column | Type | Notes |
| --- | --- | --- |
| `id` | uuid PK | Generated per contact |
| `contact_id` | uuid FK -> `contacts.id` | Unique constraint to enforce 1:1 |
| `summary` | text | Narrative paragraph (auto or user) |
| `summary_user_asserted` | boolean default false | True when user last edited |
| `summary_last_auto_update_at` | timestamptz | For telemetry |
| `published_snapshot_id` | uuid FK -> `relationship_profile_snapshots.id` | Active snapshot for message generation |
| `created_at` / `updated_at` | timestamptz | Managed by triggers |

### `relationship_profile_facts`
Purpose: normalized FORD entries with evidence references so we can store multiple facts per category, apply manual overrides, and audit how each fact was produced.

| Column | Type | Notes |
| --- | --- | --- |
| `id` | uuid PK | |
| `profile_id` | uuid FK -> `relationship_profiles.id` | |
| `category` | enum(`family`,`occupation`,`recreation`,`dreams`) | FORD grouping |
| `value` | text | Stored fact |
| `source_note_id` | uuid nullable | Present for auto facts |
| `source_excerpt` | text nullable | Short quote (<=200 chars) |
| `source_recorded_at` | timestamptz nullable | When the note was created |
| `ai_model_name` | text nullable | e.g., `gpt-4.1` |
| `ai_model_version` | text nullable | |
| `user_asserted` | boolean default false | Manual overrides |
| `updated_by` | enum(`system`,`user`) | Audit |
| `updated_at` | timestamptz | |
| Unique composite (`profile_id`,`category`) enforced only for `user_asserted=false`. For multiple facts per category, maintain ordering via `display_order int`.

### `relationship_profile_snapshots`
Purpose: immutable JSON payloads that capture the entire profile state (summary + FORD facts) whenever the system or a user change anything. Message-generation always reads from a published snapshot so it never sees in-flight edits; previous snapshots remain available for audit/history.

Stores immutable JSON used by message generation.
| Column | Type | Notes |
| `id` | uuid PK | |
| `profile_id` | uuid | |
| `snapshot_json` | jsonb | Canonical structure (summary + FORD facts + user_asserted flags + evidence) |
| `source_note_id` | uuid nullable | The triggering note |
| `ai_model_name` / `ai_model_version` | text | |
| `created_at` | timestamptz | |
| `status` | enum(`pending`,`published`,`failed`) | Message generation reads only `published` |

RLS: All tables scope by `contact.user_id = auth.uid()` via joins; snapshots should include redundant `user_id` for simpler policies.

## Workflows
### 1. Note Saved → Profile Refresh
1. Note service writes the note and emits `note_saved` event with `contact_id`.
2. Edge Function (Supabase) picks up the event:
   - Fetches current `relationship_profiles` and latest snapshot.
   - Runs AI extraction constrained to explicit mentions (e.g., regex + LLM verifying evidence).
   - Produces new facts + summary plus evidence metadata.
   - Writes updates to `relationship_profile_facts` and `relationship_profiles` within a transaction.
   - Creates a new snapshot row with status `pending` and links it to the triggering note.
3. Once DB commit succeeds, mark snapshot `published` and update `relationship_profiles.published_snapshot_id`.
4. Emit `profile_published` event for message generation; message generation jobs still rely on the last published snapshot if they started earlier.

### 2. Manual Edit
1. User taps “Edit” on a FORD fact or the summary.
2. Editing UI pre-fills current value, labels whether it’s auto or user asserted.
3. On save:
   - Write to `relationship_profile_facts` (or `relationship_profiles.summary`) with `user_asserted=true` and `updated_by='user'`.
   - Clear automated evidence fields for manual entries.
   - Regenerate snapshot (Edge Function) so message generation picks up manual overrides immediately.
4. Auto workflows respect `user_asserted` fields: they only append suggestions to `relationship_profile_suggestions` (optional future table) or leave the manual fact untouched.

### 3. Message Generation Consumption
1. When generating a birthday/event message, retrieval logic reads `relationship_profiles.published_snapshot_id`.
2. It loads the associated snapshot JSON to include summary + FORD facts in prompts.
3. Because snapshots are immutable, prompts get consistent context even if another note arrives mid-run.

## UI / UX Requirements
- Add a collapsible “Relationship Profile” card to the contact detail screen using existing contact UI components.
- Layout:
  - Summary paragraph at top with “Last refreshed {relative time}” metadata.
  - FORD displayed as four stacked sections, each showing top fact plus optional secondary bullets when multiple values exist.
  - Evidence affordance: subtle “View note” pill that opens the source note modal; hidden for user-asserted facts.
  - Manual edit affordance: “Edit” icon per section plus a global “Update summary.”
- Respect accessibility standards (44px tap targets, readable fonts, high contrast) per `standards/frontend/accessibility.md`.
- Surface `user_asserted` facts with a small badge (“Manual”) to distinguish them from AI-derived entries.
- No outdated warnings; older facts remain visible with their original timestamps.

## Evidence & Auditing Rules
- Automated entries must include `source_note_id`, `source_excerpt`, and `source_recorded_at`.
- Store `ai_model_name` + `ai_model_version` for every automated update; keep them server-side but expose via admin tooling/logs if needed.
- Evidence display remains optional: show a “View note” CTA only when `source_note_id` is present; otherwise keep UI clean.
- Logs: send structured events to Sentry/logging whenever the AI workflow publishes a snapshot, including counts of updated facts.

## Message Generation Integration
- Maintain a Supabase view `relationship_profile_context_v` that flattens published snapshots into prompt-ready fields (summary + JSON array of FORD facts, flagged by `user_asserted`).
- Message retrieval code should:
  - Query this view alongside recent notes.
  - Prefer automated facts when available, but include manual overrides when `user_asserted=true`.
- New snapshot publication triggers a lightweight webhook/event to the message generation orchestrator, but orchestrator should debounce to avoid spamming generation for rapid note edits.
- Until the new snapshot is marked `published`, message jobs continue using the previous snapshot ID for consistency.

## Error Handling & Edge Cases
- If AI parsing fails, keep the previous snapshot and surface a non-blocking toast (“Profile refresh failed; retrying”). Retry with exponential backoff via Edge Function scheduler.
- If a manual edit conflicts with an in-flight auto update, last writer wins but both operations create snapshots for audit. Implementation should consider row-level locking around `relationship_profiles` to prevent race conditions.
- Deleting a contact must cascade delete related profiles, facts, and snapshots.
- Offline usage: manual edits queue locally and sync when online; ensure conflicts merge correctly (manual overrides should generally win).

## Telemetry & Observability
- Emit events: `profile_refresh_started`, `profile_refresh_succeeded`, `profile_refresh_failed`, `profile_manual_edit`, `profile_snapshot_published`.
- Capture durations between note save and snapshot publish to monitor pipeline latency (goal <5s P95).
- Monitor count of user-asserted facts vs. auto facts to understand where AI struggles.

## Existing Code & References to Leverage
- `Assiist` reference: `/Users/benjaminmackenzie/Assiist/assiist-app-2/docs_and_scripts/rearchitecture/context_engineering/retrieval/context_retrieval_overview.md` and sibling files for how relationship profiles feed retrieval + rules matching. Align JSON schema and retrieval strategies where possible to accelerate implementation.
- Contact detail UI components already used for VIP info; extend them rather than creating a new route.
- Global standards in `agent-os/standards/global/*.md` for validation, security, and performance expectations.

## Out of Scope
- Confidence scoring, inferred facts, or hallucinated data.
- Visual redesigns beyond integrating the new card into the existing contact screen.
- Sharing/exporting profiles, analytics dashboards, or historical timelines.
- Staleness indicators or auto-hiding old facts.
- Displaying AI model metadata to end users.

## Testing Strategy
- **Unit Tests:**
  - Fact-parsing functions confirm they reject inferred data lacking explicit statements.
  - Snapshot serializer ensures `user_asserted` flags and evidence metadata persist correctly.
- **Integration Tests:**
  - Note save → workflow → snapshot pipeline (success + failure retries).
  - Manual edit syncing offline → online.
  - Message generation reading the correct snapshot while a new one is pending.
- **UI Tests:**
  - Contact screen renders FORD sections with/without evidence.
  - Accessibility checks for VoiceOver labels and tap targets.
- **Audit Tests:**
  - Verify `ai_model_name/version` stored for each auto update and retrievable for investigations.
