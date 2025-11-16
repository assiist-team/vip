# Spec Requirements: Note Signal Extraction

## Initial Description
note signal extraction

## Requirements Discussion

### First Round Questions

**Q1:** I’m assuming we’ll process each note through an LLM-powered extractor that outputs structured signals (topics, entities, sentiment) stored alongside the note. Does that match your vision, or should we run extraction in batched offline jobs?  
**Answer:** Process each note individually as it’s submitted; run the extractor one-at-a-time per note.

**Q2:** Should the first release cover the full set of signals (topics, named entities, temporal markers, interaction type, sentiment), or is it better to prioritize a smaller subset such as topics + entities + sentiment for faster delivery?  
**Answer:** Implement the complete signal taxonomy documented in `/Users/benjaminmackenzie/Assiist/assiist-app-2/docs_and_scripts/rearchitecture/context_engineering/retrieval/context_retrieval_overview.md`: `key_topics`, `note_type.primary`, `note_type.secondary[]`, `situational_markers`, `named_entities`, `temporal_signals`, `action_signals`, `location_signals`, `amounts` (optional), and `relationship_stage`. Omit sentiment entirely and keep `note_type`/interaction semantics identical to the doc.

**Q3:** I’m planning to store extracted signals in dedicated Postgres tables tied to note IDs with evidence references. Is there another storage pattern (JSONB columns on the notes table, pgvector embeddings, etc.) you’d prefer?  
**Answer:** Prefer separate JSON fields on the `notes` table for clarity; open to revisiting if that causes serious performance issues.

**Q4:** For the extraction pipeline, should we call the LLM from a Supabase Edge Function per note submission, or would you rather queue requests (e.g., via cron job + status tracking) to control cost spikes?  
**Answer:** Call the LLM from a Supabase Edge Function on each submission.

**Q5:** I’m assuming we need guardrails for privacy and hallucination control—e.g., enforce deterministic schemas, confidence scores, and flag low-confidence outputs for review. Are there additional validation rules or human-in-the-loop checks you want?  
**Answer:** Enforce the same evidence-bound approach as the referenced project: strict schemas, confidence scores, cite-as-you-say evidence tying each signal back to note content.

**Q6:** Do we need any real-time feedback in the Flutter app (e.g., “signals processing” status per note), or is it acceptable for processing to happen silently in the background?  
**Answer:** No real-time UI feedback needed; processing can run silently with later push notifications when AI outputs are ready.

**Q7:** Are there any specific exclusions or future-phase items (like auto-tagging contacts or triggering notifications) that we should keep out of scope for this spec?  
**Answer:** Keep the focus on extraction; downstream notifications already handled elsewhere, so no additional real-time interactions or auto-tagging in this scope.

### Existing Code to Reference
**Similar Features Identified:**
- Feature: Prior signal extraction pipeline (retrieval context) – Path: `/Users/benjaminmackenzie/Assiist/assiist-app-2/docs_and_scripts/rearchitecture/context_engineering/retrieval/`
- Feature: Task generation spec (evidence-bound patterns) – Path: `/Users/benjaminmackenzie/Assiist/assiist-app-2/docs_and_scripts/rearchitecture/context_engineering/task_generation_spec_v2.md`

### Follow-up Questions
**Follow-up 1:** You mentioned the previous project defined the exact set of signals. Could you confirm which signal categories from those docs we must support for this first release (e.g., topics, named entities, etc.) so we implement the same scope here?  
**Answer:** Confirmed—mirror the `phase_1_signal_extraction` spec in `/Users/benjaminmackenzie/Assiist/assiist-app-2/docs_and_scripts/rearchitecture/context_engineering/retrieval/context_retrieval_overview.md`, capturing `key_topics`, `note_type.primary`, `note_type.secondary[]`, `situational_markers`, `named_entities` (with type/role/status), `temporal_signals`, `action_signals`, `location_signals`, optional `amounts`, and `relationship_stage`.

**Follow-up 2:** Since we’ll store signals as JSON on each note record, what schema shape do you expect—single `signals` object with arrays per category, or separate JSON fields (e.g., `topics`, `entities`)? Any limits on payload size we should enforce?  
**Answer:** Prefer separate JSON fields per signal category; no payload limits identified.

**Follow-up 3:** For the “cite-as-you-say” evidence requirement, do we need to store explicit references such as note spans/timestamps plus confidence scores per extracted item, or is a note-level reference sufficient?  
**Answer:** Follow the documentation’s evidence model; note-level references with cite-as-you-say linkage are sufficient for this scope.

## Visual Assets

### Files Provided:
No visual assets provided.

## Requirements Summary

### Functional Requirements
- Run structured signal extraction for every submitted note immediately via Supabase Edge Function.
- Extract the relationship-focused signal set (derived from `context_retrieval_overview.md` but simplified for VIP) described below—topics, interaction purpose, named entities, temporal signals, and supporting details—omitting sentiment and other unused fields.
- Store each signal category as a dedicated JSON field on the `notes` table, preserving cite-as-you-say evidence and confidence scores aligned with the legacy spec.
- Ensure extraction runs silently in the background; downstream systems trigger push notifications when later stages (e.g., message generation) finish.

### Signal Schema Decisions
- `key_topics.items[]` — 5-10 noun-phrase themes summarizing note content; strictly “what was discussed.”
- `interaction_purpose` — `{ primary: enum, secondary[]: enum }` using a deterministic intent taxonomy (e.g., `follow_up_required`, `life_update`, `event_recap`). No extra flags; urgency/tone encoded via enum choices.
- `named_entities.items[]` — `{ name: string, type: enum(person|org|project|other), contact_context: enum(primary_contact|contact_related|external), role?: string }` to capture who’s involved without conflating relationships to the user.
- `temporal_signals.items[]` — `{ surface_text: string, normalized_iso: string, specificity: enum(day|week|month|relative), confidence: float }` holding every explicit time reference.
- `supporting_details.items[]` — `{ detail_type: enum(action|location|amount), text: string, normalized_value?: string|number, related_entities?: string[] }` consolidating actionable facts (send/update/meet/payment) in one orthogonal bucket.
- `evidence` — every list item above embeds `{ note_reference: string, spans?: [{start:int,end:int}], confidence: float }`, ensuring cite-as-you-say provenance per signal entry rather than coarse note-level references.

### Reusability Opportunities
- Mirror the prior Assiist context-engineering retrieval pipeline for signal definitions and evidence handling.
- Reuse the evidence-bound practices from `task_generation_spec_v2.md` for cite-as-you-say and confidence scoring patterns.

### Scope Boundaries
**In Scope:**
- Per-note extraction pipeline invoked on submission.
- JSON persistence of each signal class with provenance and confidence metadata.
- Schema validation and hallucination guardrails mirroring the legacy project.

**Out of Scope:**
- UI feedback loops or progress indicators in the Flutter app.
- Auto-tagging contacts or triggering new notification flows.
- Sentiment analysis or additional signal types beyond those listed in the legacy documentation.

### Technical Considerations
- Supabase Edge Function per note submission will call the LLM (`gpt-4o-mini` equivalent) at temperature 0 with structured schema prompts.
- JSON storage on the `notes` table avoids additional tables but must be monitored for performance; revisit if query latency becomes problematic.
- Cite-as-you-say requires storing note-level references plus confidence scores; align with prior implementation guidance for evidence binding.


