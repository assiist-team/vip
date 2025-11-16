# Specification: Note Signal Extraction

## Goal
Extract the minimum set of orthogonal, relationship-focused signals from every note so downstream context retrieval and message generation can reason about *what happened*, *why it matters*, *who was involved*, *when it matters*, and any actionable details—all with cite-as-you-say provenance. The pipeline must run automatically on each note submission, store results as JSON on the `notes` record, and expose retry/audit metadata without blocking the user experience.

## User Stories
- As a VIP user, when I save a note I want signals captured automatically so future AI messages reference the right people, topics, and timing without extra data entry.
- As the context retrieval service, I want a deterministic schema per note so I can build prompts and retrieval queries without re-running heavy extraction.
- As the operations team, I need visibility into extraction status and evidence so I can trace mistakes and rollback bad outputs.

## Dependencies & Contracts
- **Basic Notes System (`2025-11-16-basic-notes-system`)**: Emits `notes` records; exposes Supabase trigger or Edge Function hook to start extraction.
- **Context Retrieval & Task Generation (Assiist reference docs)**: Consumes the `topics/purpose/entities/time/supporting_details` JSON fields; requires cite-as-you-say evidence to trust signals.
- **Push Notifications / Message Generation specs (future)**: Assume signals exist by the time a message is scheduled; failures here must surface errors so downstream jobs can retry with fallback prompts.
- **Global Standards**: Respect validation, security, and coding standards under `agent-os/standards/global/*`.

## Functional Overview
1. **Note saved**: Flutter client writes note via Supabase RPC/insert.
2. **Edge Function trigger**: Supabase function `notes_signal_extractor` (Python) receives note payload (id, contact_id, text, language).
3. **LLM extraction**: Function calls `gpt-4o-mini` with a structured JSON schema prompt (temperature 0) as defined below.
4. **Validation & guardrails**:
   - JSON schema validation (pydantic) enforces required fields.
   - Evidence validation ensures every list item contains cite-as-you-say references.
   - Confidence thresholds (<0.5) force the item into a `rejected` list for auditing.
5. **Persistence**:
   - Update the originating `notes` row with five JSON fields plus metadata columns (`signals_status`, `signals_version`, `signals_processed_at`, `signals_error`).
   - Append an immutable row in `note_signal_audit` for traceability.
6. **Downstream notification**: Publish `note_signals_ready` event (Supabase channel) so context retrieval/jobs can proceed asynchronously.
7. **Retries**: Failure updates `signals_status='error'` and leaves existing signals untouched; scheduled worker retries with exponential backoff.

## Signal Schema
Each top-level bucket is stored as a JSON column on `notes` with the exact shapes below. All arrays cap at 10 items; extractor truncates deterministically.

### `signal_topics` (`jsonb`)
```json
{
  "items": [
    {
      "text": "timeline concerns",
      "evidence": {
        "note_reference": "note:2025-11-16#1",
        "spans": [{"start": 54, "end": 74}],
        "confidence": 0.82
      }
    }
  ]
}
```
- Max 10 noun-phrase themes.
- Names cannot duplicate interaction purpose labels.

### `signal_interaction_purpose` (`jsonb`)
```json
{
  "primary": "follow_up_required",
  "secondary": ["life_update"],
  "evidence": {
    "note_reference": "note:2025-11-16#1",
    "spans": [{"start": 0, "end": 25}],
    "confidence": 0.78
  }
}
```
- `primary` required, `secondary[]` optional.
- Enum catalog maintained in code (`purpose_taxonomy.json`).
- No extra booleans; urgency/tone conveyed through enum name (e.g., `urgent_follow_up`).

### `signal_entities` (`jsonb`)
```json
{
  "items": [
    {
      "name": "Sarah Chen",
      "type": "person",
      "contact_context": "primary_contact",
      "role": "client",
      "evidence": {
        "note_reference": "note:2025-11-16#1",
        "spans": [{"start": 5, "end": 15}],
        "confidence": 0.9
      }
    },
    {
      "name": "Brandon",
      "type": "person",
      "contact_context": "contact_related",
      "role": "spouse",
      "evidence": { "...": "..." }
    }
  ]
}
```
- `contact_context ∈ {primary_contact, contact_related, external}` clarifies relationship without referencing the user.
- Optional `role` string (<40 chars).

### `signal_temporal` (`jsonb`)
```json
{
  "items": [
    {
      "surface_text": "this Friday",
      "normalized_iso": "2025-11-21",
      "specificity": "day",
      "confidence": 0.85,
      "evidence": { "...": "..." }
    }
  ]
}
```
- `specificity ∈ {day, week, month, relative}`; relative keeps `normalized_iso=null`.

### `signal_supporting_details` (`jsonb`)
```json
{
  "items": [
    {
      "detail_type": "action",
      "text": "Send updated budget",
      "normalized_value": null,
      "related_entities": ["Sarah Chen"],
      "evidence": { "...": "..." }
    },
    {
      "detail_type": "amount",
      "text": "$20k retainer",
      "normalized_value": 20000,
      "related_entities": ["Project Falcon"],
      "evidence": { "...": "..." }
    }
  ]
}
```
- `detail_type ∈ {action, location, amount}`; more types can be added via taxonomy file.

### Evidence Structure
All evidence payloads share the same structure:
```json
{
  "note_reference": "note:YYYY-MM-DD#ordinal",
  "spans": [{"start": 0, "end": 42}],
  "confidence": 0.0 - 1.0
}
```
- Spans optional but recommended when the note text is stored.
- Confidence computed from LLM self-report (bounded) combined with deterministic heuristics (e.g., presence of explicit entities).

## Data Model Changes

### `notes` (existing table)
Add columns (jsonb unless noted):
- `signal_topics`
- `signal_interaction_purpose`
- `signal_entities`
- `signal_temporal`
- `signal_supporting_details`
- `signals_status` (enum: `pending`, `processing`, `ready`, `error`)
- `signals_version` (text) – tracks prompt/schema version.
- `signals_processed_at` (timestamptz)
- `signals_error` (text nullable) – last failure code/message.

RLS inherits from existing `notes` policies; new columns piggyback on same permissions.

### `note_signal_audit`
| Column | Type | Notes |
| --- | --- | --- |
| `id` | uuid PK | |
| `note_id` | uuid FK -> `notes.id` | |
| `signals_payload` | jsonb | Full response exactly as written |
| `status` | enum(`ready`,`rejected`,`error`) | |
| `model_name` | text | e.g., `gpt-4o-mini` |
| `model_version` | text | |
| `prompt_version` | text | |
| `error_message` | text nullable | |
| `created_at` | timestamptz default now() | |

Purpose: immutable log for debugging; also used to restore older signals if needed.

## Edge Function Flow (`notes_signal_extractor`)
1. **Trigger**: Supabase database trigger on `notes` inserts/updates sets `signals_status='pending'` and invokes Edge Function via webhook payload.
2. **Debounce guard**: If `signals_status` already `processing` or `ready` for the current `signals_version`, skip work (prevents duplicate runs).
3. **LLM prompt**: Provide note text, contact metadata (first name, closeness), and taxonomy definitions. Response must conform to JSON schema described above.
4. **Validation**:
   - Schema validation using Pydantic models mirroring the JSON definitions.
   - Evidence check ensures `note_reference` matches note ID and spans fall within note length.
   - Deduplicate topics/entities by lowercased text.
5. **Persistence**:
   - Within a single transaction: update `notes` columns, set `signals_status='ready'`, and write audit row.
6. **Post-commit**:
   - Publish `note_signals_ready` event (payload: note_id, contact_id).
   - Emit telemetry log to Sentry with status metrics.

## Error Handling & Retries
- Any validation failure sets `signals_status='error'` and populates `signals_error`.
- Cloud Task (or Supabase cron) re-queues notes in `error` state up to 3 times with exponential backoff (15s, 2m, 10m). After final failure, raise alert.
- Partial successes (e.g., some items rejected) still mark status `ready`; rejected items are recorded in `note_signal_audit` for inspection but not stored on the note.

## Performance & SLAs
- Target extraction latency: < 1.5s P95 per note (Edge Function runtime + LLM call).
- Throughput: must handle burst of 30 notes/min per user; Edge Function should be stateless and horizontally scalable.
- Payload sizes: limit each JSON column to ~10 KB; enforce truncation heuristics to avoid oversized records.

## Security & Privacy
- Edge Function runs with service-role key; validate `user_id` against note ownership before processing.
- Redact sensitive data from logs except hashed note_id references.
- All evidence references stay within the originating user’s namespace to maintain RLS guarantees.

## Telemetry & Observability
- Emit structured logs: `signals_processing_started`, `signals_processing_succeeded`, `signals_processing_failed`.
- Metrics to track:
  - Latency (start → ready).
  - Failure reasons (LLM rate limit, validation, Supabase write).
  - Average confidence per bucket to detect drift.
  - Count of items per bucket (should stay within expected range).

## Testing Strategy
- **Unit Tests**: Pydantic schema validation, evidence validator, dedupe logic.
- **Integration Tests**: Simulated note insert → Edge Function → Supabase update; include retry paths.
- **Contract Tests**: JSON schema shared with downstream context retrieval to ensure compatibility.
- **Load Tests**: Fire 100 concurrent extraction requests to confirm service elasticity and Supabase write contention.

## Out of Scope
- Sentiment analysis or additional signal buckets beyond the five defined here.
- UI indications of processing status (handled by later specs).
- Automatic tasks/notifications triggered directly from signal extraction.

## References
- Requirements: `agent-os/specs/2025-11-16-note-signal-extraction/planning/requirements.md`
- Legacy signal spec: `/Users/benjaminmackenzie/Assiist/assiist-app-2/docs_and_scripts/rearchitecture/context_engineering/retrieval/context_retrieval_overview.md`
- Guardrail inspiration: `.../task_generation_spec_v2.md`

