# Spec Requirements: Evidence-Based Relationship Profiles

## Initial Description
Evidence-Based Relationship Profiles — Create contact-level profile system with FORD categories and relationship summary fields that update incrementally from notes, maintaining evidence references (which notes support each piece of information) and timestamps for transparency.

## Requirements Discussion

### First Round Questions

**Q1:** I’m assuming each VIP contact gets a dedicated “Relationship Profile” screen with FORD sections plus a concise narrative summary at the top—should we default to that layout, or do you prefer a different grouping?
**Answer:** Keep FORD plus the relationship summary on the same view as the contact’s existing details; the layout described is fine so long as it coexists with the basic contact info.

**Q2:** I’m thinking profile data updates happen automatically whenever a new note is saved (via incremental parsing) and users can optionally edit fields—do we allow manual overrides, or should entries remain read-only evidence snapshots?
**Answer:** Every new note should trigger the AI workflow that refreshes the full profile and also kicks off message generation. Users may edit fields manually, while automatic updates must retain evidence links.

**Q3:** For evidence tracking, I’m planning to store note IDs, excerpts, and timestamps alongside every FORD fact and summary sentence so users can tap to view the source note—do we also need to capture which model/version produced the inference for auditing?
**Answer:** Surface evidence subtly—no heavy emphasis on note IDs/excerpts in the UI—but still store them. Capture model/version metadata for auditing, though it doesn’t need to be user-visible.

**Q4:** To keep information trustworthy, I’m assuming we only persist facts when the note explicitly states them (no inference) and we keep the latest two supporting notes for each fact—should we relax that to allow inferred data with confidence scores?
**Answer:** Do not allow inferred facts or confidence scoring; only explicit statements with evidence should be persisted.

**Q5:** Relationship summaries: should the system maintain multiple short summaries (e.g., “Interaction history,” “Opportunities,” “Risks”) or just a single paragraph? If multiple, which sections matter most to you?
**Answer:** Use a single paragraph-style field (possibly multiple paragraphs within it) so the output reads naturally for humans.

**Q6:** Evidence freshness: do we hide/flag facts when their supporting notes are older than a configurable threshold (e.g., 12 months), or should everything stay visible with an “outdated” badge?
**Answer:** Keep older facts visible without hiding or applying outdated badges.

**Q7:** Collaboration with other features: should this profile feed data back into message generation immediately (making fields queryable via Supabase views), or is it purely for user-facing browsing in this first release?
**Answer:** It must feed data back into message generation as already outlined in prior docs.

**Q8:** Are there any specific details or behaviors you definitely do NOT want included in this initial scope (e.g., analytics, editing history, sharing/export)?
**Answer:** No extra exclusions noted, but prior documentation (e.g., `Assiist` context-engineering materials) shows how profiles drive retrieval; reference those patterns.

### Existing Code to Reference

**Similar Features Identified:**
- Feature: Relationship profile usage patterns – Path: `/Users/benjaminmackenzie/Assiist/assiist-app-2/docs_and_scripts/rearchitecture/context_engineering/retrieval/context_retrieval_overview.md`
- Additional Context: Broader `/Users/benjaminmackenzie/Assiist/assiist-app-2/docs_and_scripts/rearchitecture/context_engineering/` directory for related workflows

### Follow-up Questions

**Follow-up 1:** When a user manually edits a FORD fact or the summary paragraph, should we require them to choose a supporting note, or can we tag those edits as “user asserted” without evidence?
**Answer:** Tag manual edits as user-asserted without requiring evidence selection.

**Follow-up 2:** Each new note kicks off both the relationship-profile refresh and message-generation workflow. Should message generation use the freshly updated profile immediately, or wait until manual edits/reviews finish?
**Answer:** Message generation should use the previously published profile snapshot; new notes can trigger generation but shouldn’t rely on the just-updated profile until it’s confirmed.

## Visual Assets

No visual assets provided.

## Requirements Summary

### Functional Requirements
- Display FORD sections and a narrative summary directly on each contact’s main screen alongside core contact info.
- Automatically refresh the entire profile plus trigger message-generation workflows whenever a note is saved.
- Allow manual edits to FORD fields and summary, marking them as user-asserted entries.
- Store and subtly surface evidence links (note references, timestamps) for auto-generated data; manual edits may omit evidence.
- Persist only explicitly stated facts—no inferred data or confidence scores.
- Provide auditing metadata (model/version) internally for all automated profile updates.
- Ensure message-generation pipelines query the established profile dataset even as updates are pending.

### Reusability Opportunities
- Reference the Assiist context-engineering retrieval patterns for how relationship profiles power querying logic and rule selection.
- Reuse existing contact detail UI components to host the FORD sections and summary instead of building a new screen.

### Scope Boundaries
**In Scope:**
- Integrating profile data with the main contact view.
- Automated refresh and evidence capture tied to note saves.
- Manual editing with user-asserted tagging.
- Feeding curated profiles into message-generation services using existing retrieval patterns.

**Out of Scope:**
- Hiding/flagging stale facts or applying outdated badges.
- Exposing auditing metadata (model/version) in the UI.
- Confidence scoring, inference-based fact creation, or analytics/sharing tooling for profiles.

### Technical Considerations
- Leverage Supabase views/tables so message generation can query stable profile data; updates should be transactional to avoid inconsistent reads.
- AI workflow must log model/version per update for auditability.
- Maintain evidence references in storage while keeping UI presentation minimal; consider expandable sections for detailed provenance.
- Ensure sequencing so message-generation jobs use the last confirmed profile snapshot, not speculative updates.
- Align terminology and structures with the Assiist context-engineering documentation for easier future reuse.
