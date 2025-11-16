# Tasks: Note Signal Extraction

## 1. Data Model & Storage
- **Create migrations** adding JSON signal columns and status metadata to `notes` plus the `note_signal_audit` table with required indexes and enums.
- **Update Supabase RLS policies** (if needed) to ensure new columns inherit existing access controls without exposing other users’ signals.
- **Document column schemas** (topic, interaction purpose, entities, temporal, supporting details, evidence) for developers and LLM prompt references.

## 2. Edge Function Foundation
- ** scaffold `notes_signal_extractor`** Supabase Edge Function (Python) with dependency wiring (HTTP handler, Supabase client, OpenAI/Anthropic SDK placeholders).
- **Implement secure invocation**: validate payload, fetch note + contact context, short-circuit if already processed for current `signals_version`.

## 3. LLM Extraction Pipeline
- **Design structured prompt** reflecting the simplified schema and deterministic enums (purpose taxonomy, detail types, entity context options).
- **Implement call wrapper** around `gpt-4o-mini` (or configured model) with retries, timeouts, and token budgeting.
- **Build schema validator** (Pydantic models) enforcing per-item evidence, array caps, and enum membership; reject or trim invalid entries deterministically.

## 4. Persistence & Auditing
- **Transform validated payload** into the five JSON columns plus metadata (`signals_status`, `processed_at`, `version`, `error`).
- **Write transactional update** that stores note columns and inserts an immutable `note_signal_audit` row; handle optimistic concurrency.
- **Publish `note_signals_ready` event** after commit so downstream retrieval jobs can subscribe.

## 5. Error Handling & Retries
- **Implement failure paths** that set `signals_status='error'`, capture `signals_error`, and push telemetry breadcrumbs.
- **Create scheduled retry job** (Supabase cron/Edge Function) that requeues notes stuck in `error`, respecting exponential backoff and attempt caps.

## 6. Telemetry & Observability
- **Emit structured logs/metrics** (`signals_processing_started/succeeded/failed`) including durations, counts per bucket, and model versions.
- **Add alerting hooks** (Sentry or Supabase logs) when error rate or latency exceeds thresholds.

## 7. Testing Strategy
- **Unit tests** for schema validation, deduplication, evidence span checks, and enum mapping.
- **Integration tests** simulating note insert → Edge Function → Supabase updates (success, validation failure, retry).
- **Load test script** to fire concurrent extraction requests ensuring the function scales and DB writes stay within SLAs.

## 8. Documentation & Handover
- **Update developer docs** (README/spec appendices) detailing signal JSON shapes, taxonomy files, and how downstream services consume them.
- **Create runbook** covering failure troubleshooting, manual reprocessing, and audit log usage.

