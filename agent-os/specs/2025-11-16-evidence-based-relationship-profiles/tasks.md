# Tasks: Evidence-Based Relationship Profiles

## 1. Data Modeling & Database Foundations
- Create `relationship_profiles`, `relationship_profile_facts`, and `relationship_profile_snapshots` tables with RLS policies scoped by user ownership through contacts.
- Define enums/types (`ford_category`, `snapshot_status`) and necessary indexes/unique constraints.
- Implement triggers to maintain `updated_at`, enforce single profile per contact, and cascade deletes when contacts are removed.
- Build Supabase view `relationship_profile_context_v` that flattens published snapshots for retrieval pipelines.

## 2. AI Workflow & Edge Functions
- Implement Edge Function that listens for `note_saved` events, loads existing profile data, and invokes the AI extraction process.
- Constrain extraction to explicit statements; validate outputs before persistence (reject inferred facts).
- Write transactional logic to update facts, summary, evidence metadata, and snapshot rows; handle retries and failure states.
- Emit structured logs/telemetry (`profile_refresh_*` events) and capture model/version metadata per update.

## 3. Manual Edit Experience
- Extend contact detail UI with edit affordances for each FORD section and summary field.
- Build forms that distinguish auto vs. user-asserted entries, tag manual edits appropriately, and clear evidence fields.
- Handle offline edit queueing and sync resolution, ensuring manual overrides win when conflicts occur.
- Trigger snapshot regeneration after manual edits so downstream consumers read the latest overrides.

## 4. Contact Detail UI Integration
- Add collapsible “Relationship Profile” card to the contact view, reusing existing components for layout consistency.
- Display summary with “Last refreshed” metadata and badges indicating manual entries.
- Show FORD facts with subtle evidence affordances (“View note”) when source data exists; hide for user-asserted entries.
- Ensure accessibility compliance (tap targets, contrast, VoiceOver labels) and responsiveness.

## 5. Snapshot Publication & Message Generation Contract
- Implement snapshot publishing flow (`pending` → `published`) with atomic updates to `relationship_profiles.published_snapshot_id`.
- Debounce message-generation triggers so rapid note saves don’t spam downstream jobs.
- Ensure message generation queries the published snapshot via the new view and gracefully handles pending/failed states.

## 6. Error Handling, Observability, and Testing
- Add Sentry logging for workflow successes/failures and manual edits; include latency metrics between note save and publish.
- Implement retry/backoff for failed AI runs and surface non-blocking UI notifications when refreshes fail.
- Create automated tests: unit (parsers, serializers), integration (note→profile pipeline, manual edit sync), UI (rendering, accessibility), and audit verifications (model metadata persisted).
- Document operational runbooks for monitoring profile refresh latency and troubleshooting stuck snapshots.
