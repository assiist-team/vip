# VIP Context Engineering — Technical Specification

## Overview

This document provides technical implementation guidance for VIP's context engineering system. It describes how the app processes user notes, maintains relationship context, retrieves relevant information, and generates personalized messages.

**Design Philosophy:**
- **Simpler than predecessor**: VIP focuses on message generation (not complex task management)
- **Evidence-bound**: Profile information and message content must trace back to actual notes
- **Pattern-based**: Rules and examples match on functional patterns, not surface features
- **Count-based retrieval**: "Last N notes" is more meaningful than time windows for varying interaction frequencies

---

## Architecture Overview

```
User logs note
    ↓
1. Signal Extraction (LLM-based)
   → Extract: key_topics, entities, temporal signals, interaction type, sentiment
    ↓
2. Profile Update (evidence-based)
   → Update: FORD fields, relationship summary, relationship metrics
   → Each update includes evidence references and timestamps
    ↓
3. Context Retrieval (when message trigger detected)
   → Gather: Recent notes (count-based), relevant historical notes (vector search), 
            relationship profile, past messages
    ↓
4. Rule Retrieval (pattern-based)
   → Anonymize entities → Build query → Retrieve matching rules
    ↓
5. Message Generation
   → Input: Current context + retrieved rules + relationship profile
   → Output: Personalized message draft
    ↓
6. User Review & Send
   → User revises if needed → Send via native messaging app
```

---

## Component 1: Signal Extraction

**Purpose:** Extract structured information from notes to guide downstream processing.

### Input
- `raw_note` (string): The note text
- `contact_id` (string): Contact identifier
- `current_datetime` (ISO string): Timestamp
- `contact_name` (optional string): For entity resolution

### Output (Signals Bundle)
```json
{
  "key_topics": ["family_health", "project_discussion", "personal_check_in"],
  "note_type": {
    "primary": "follow_up_required",
    "secondary": ["social_interaction"]
  },
  "named_entities": [
    {"text": "Sarah", "type": "person", "context": "primary contact"},
    {"text": "John", "type": "person", "context": "spouse present"},
    {"text": "website redesign", "type": "project", "status": "active"}
  ],
  "temporal_signals": [
    {"text": "this Friday", "interpretation": "2024-11-22", "specificity": "day", "confidence": "high"}
  ],
  "interaction_type": "in_person_meeting",
  "sentiment_indicators": ["positive", "concerned"],
  "urgency_level": "medium"
}
```

### Implementation Notes

**Model Selection:**
- Use fast, inexpensive model (e.g., `gpt-4o-mini` or `claude-3-haiku`)
- Temperature: 0.0 (maximize consistency)
- Expected latency: 300-500ms
- Expected cost: ~$0.0001-0.0003 per note

**Key Principles:**
1. **Extract only what's explicitly mentioned** — no inference or assumptions
2. **Prioritize key_topics** — these are the primary signals for retrieval and rule matching
3. **Keep signals orthogonal** — avoid duplicating information across fields
4. **Named entities include projects** — not just people and organizations

**Usage:**
- Signals are **hints for decision-making**, not user-facing data
- Used for: profile updates, message triggers, rule retrieval, context retrieval
- Never shown directly to users

---

## Component 2: Evidence-Based Profile Updates

**Purpose:** Maintain accurate, trustworthy relationship profiles grounded in actual interactions.

### Profile Structure

```json
{
  "contact_id": "c_123",
  "ford_categories": {
    "family": {
      "value": "Married to John; two kids in elementary school",
      "evidence": {
        "note:2024-11-01#1": "Mentioned John and kids at lunch",
        "note:2024-10-15#3": "Kids' soccer practice on weekends"
      },
      "last_updated_at": "2024-11-01T14:30:00Z"
    },
    "occupation": {
      "value": "Senior consultant at Acme Corp",
      "evidence": {
        "note:2024-09-20#2": "Started new role at Acme"
      },
      "last_updated_at": "2024-09-20T09:00:00Z"
    },
    "recreation": {
      "value": "Plays tennis; enjoys hiking",
      "evidence": {
        "note:2024-10-05#1": "Tennis tournament this weekend"
      },
      "last_updated_at": "2024-10-05T16:00:00Z"
    },
    "dreams": {
      "value": "Wants to start own consulting firm",
      "evidence": {
        "note:2024-08-12#4": "Discussed entrepreneurship goals"
      },
      "last_updated_at": "2024-08-12T11:00:00Z"
    }
  },
  "relationship_summary": {
    "value": "Met through jiu jitsu; now collaborating on referrals. Regular monthly lunches.",
    "evidence": {
      "note:2024-11-01#1": "Monthly lunch tradition",
      "note:2024-07-10#2": "Started referral arrangement"
    },
    "last_updated_at": "2024-11-01T14:30:00Z"
  },
  "relationship_metrics": {
    "relationship_age": {
      "value": "known each other for 3 years",
      "evidence": {
        "note:2024-06-15#3": "Met 3 years ago at jiu jitsu gym"
      },
      "last_evidence_at": "2024-06-15T10:00:00Z"
    },
    "interaction_rhythm": {
      "value": "monthly cadence, last contact 0 days ago",
      "evidence": {
        "note:2024-11-01#1": "Monthly lunch today",
        "note:2024-10-01#1": "Previous monthly lunch",
        "note:2024-09-05#2": "Quick phone check-in"
      },
      "last_evidence_at": "2024-11-01T14:30:00Z"
    },
    "closeness_level": {
      "value": 5,
      "note": "References Section 5 scale (1-7). User-adjustable. Communication style derived during generation."
    }
  }
}
```

### Update Policy

**Trigger:** Profile updates happen when a new note is logged.

**Update Rules:**
1. **Only update fields with explicit support from the current note**
2. **Preserve existing information unless new note supersedes it**
3. **Add evidence references for all new/updated claims**
4. **Update timestamps to reflect when information was last confirmed**
5. **Conservative approach:** When in doubt, don't update

**Example Update Flow:**
```
User logs note: "Had lunch with Sarah. She mentioned her daughter just started high school."

Signal Extraction identifies:
- key_topics: ["family_update"]
- named_entities: [{"text": "Sarah", "type": "person"}, {"text": "daughter", "type": "person", "context": "family member"}]

Profile Update:
- FORD.family: Add/update "Daughter in high school"
- evidence: {"note:2024-11-01#1": "Daughter just started high school"}
- last_updated_at: "2024-11-01T14:30:00Z"
```

**Token Budget:**
- Keep profile fields concise (aim for 1-2 sentences per field)
- Maximum ~300 tokens for entire profile
- Prioritize recent, well-supported information when trimming

---

## Component 3: Context Retrieval

**Purpose:** Gather relevant information when generating a message.

### Retrieval Strategy: Seeds + Vector Search

**Approach 1: Count-Based Seeds (Deterministic)**
- **Recent notes**: Last N notes for the contact (default: N=3)
  - Why count-based? "Last 3 notes" is interpretable regardless of interaction frequency
  - Contrast: "Last 30 days" is arbitrary and varies by relationship
- **Past messages**: Last M messages sent to this contact (default: M=5)
  - Avoid repetition and track conversation flow
- **Relationship profile**: Full FORD + summary + metrics
  - Used as hints for tone and context

**Approach 2: Vector Retrieval (Semantic Search)**
- **Purpose**: Find older relevant notes that inform current message
- **Query construction**: Uses extracted signals (key_topics, entities, temporal signals)
- **Search scope**: All historical notes for the contact
- **Retrieval limit**: Top K results (default: K=8)
- **Gating threshold**: Minimum similarity score (e.g., 0.80) to ensure relevance

### Example Retrieval Scenario

```
Current note: "Saw Sarah today. Asked about her mother's surgery, which she's recovering well from."

Signals extracted:
- key_topics: ["family_health", "social_interaction"]
- named_entities: [{"text": "Sarah", "type": "person"}, {"text": "mother", "type": "person", "context": "family member"}]

Seeds retrieved:
- Last 3 notes about Sarah
- Last 5 messages sent to Sarah

Vector search query: "family health mother surgery Sarah"
→ Retrieves older notes mentioning Sarah's mother's health issues
→ Provides context: "Her mother had hip replacement scheduled for October"

Combined context enables message like:
"Hey Sarah - good seeing you today. Was glad to hear your mom's recovery is going well."
```

### Implementation Notes

- **Parallel execution**: Seeds and vector search can run simultaneously
- **Deduplication**: If a note appears in both seeds and vector results, include it once
- **Ordering**: Organize by type (notes, messages), then by recency within type
- **Token budget**: Target ~1000-1500 tokens for all retrieved context

---

## Component 4: Rule Retrieval (Pattern-Based Matching)

**Purpose:** Retrieve situation-specific guidance for message generation.

### Entity Anonymization for Pattern Matching

**Problem:** Embedding models can create false positives based on specific names.
- "Had lunch with Sarah" might match examples mentioning "Sarah" rather than functional patterns

**Solution:** Anonymize entities before querying rule database.
- "Had lunch with Sarah and John" → "Had lunch with [CONTACT] and [CONTACT]'s spouse"
- Embeddings now match on functional patterns (multi-person interaction) not names

### Anonymization Algorithm

```python
def anonymize_for_rule_matching(note, signals):
    """Replace entity names with type placeholders."""
    anonymized = note
    
    for entity in signals["named_entities"]:
        if entity["type"] == "person":
            anonymized = anonymized.replace(entity["text"], "[CONTACT]")
        elif entity["type"] == "organization":
            anonymized = anonymized.replace(entity["text"], "[ORG]")
        elif entity["type"] == "project":
            anonymized = anonymized.replace(entity["text"], "[PROJECT]")
    
    return anonymized
```

### Query Construction

Combine anonymized note + extracted signals + relationship context:

```
Query components:
1. Anonymized note: "Had lunch with [CONTACT] and [CONTACT]'s spouse. Discussed [PROJECT]."
2. Relationship context: "Known [CONTACT] for 3 years. Professional relationship with personal rapport."
3. Extracted signals: "multi_person_meeting deliverable_owed timeline_mentioned follow_up_required"

Final query (concatenated):
"Had lunch with [CONTACT] and [CONTACT]'s spouse. Discussed [PROJECT]. Known [CONTACT] 
for 3 years. Professional relationship with personal rapport. multi_person_meeting 
deliverable_owed timeline_mentioned follow_up_required"
```

### Rule Structure

Rules are stored generically with abstract pattern descriptions:

```json
{
  "rule_id": "rule_multi_person_001",
  "situation": "Primary contact and spouse/partner both present at interaction. Established relationship.",
  "relationship_context": "Professional relationship with personal rapport. Regular contact.",
  "signals": {
    "key_topics": ["multi_person_meeting"],
    "interaction_type": "in_person",
    "relationship_stage": "established"
  },
  "good_example": "Good seeing you both yesterday. Thanks for taking the time to meet.",
  "bad_example": "Great meeting you today. I'll follow up soon.",
  "why": "Acknowledge all parties when multiple people were present. Use past tense ('yesterday') not generic present. Established relationships don't need formal 'great meeting you' language.",
  "tags": ["multi_person", "acknowledgment", "established_relationship"]
}
```

### Retrieval Process

1. **Anonymize** current note + relationship profile using extracted entities
2. **Combine** with extracted signals (key_topics, interaction_type, etc.)
3. **Generate embedding** for the query
4. **Search** rule database (vector collection)
5. **Filter** by user_id and optional signal tags
6. **Retrieve** top K rules (default: K=3-5)
7. **Return** rules with similarity scores

### Usage in Message Generation

Retrieved rules guide both "what to say" and "how to say it":
- **What**: Situation pattern indicates message is warranted
- **How**: Good/bad examples show phrasing, tone, structure
- **Why**: Explanation helps model understand reasoning

---

## Component 5: Message Generation

**Purpose:** Create personalized, contextually appropriate message drafts.

### Input Components

1. **Current note** (raw text)
2. **Retrieved context** (recent notes, relevant historical notes, past messages)
3. **Relationship profile** (FORD categories, relationship summary, metrics)
4. **Retrieved rules** (situation-specific guidance patterns)
5. **User communication style** (tone examples provided by user)
6. **Closeness scale** (numeric indicator of formality/familiarity)

### Generation Principles

**Evidence-Bound:**
- Reference actual events and information from notes
- Don't invent facts or assume details not in context

**Tone-Appropriate:**
- Match closeness scale (1=formal, 7=very familiar)
- Use relationship metrics as hints (age, rhythm, closeness level)
- Derive communication style from closeness level (not stored separately)
- Follow user's communication style examples

**Context-Aware:**
- Acknowledge multi-person interactions (mention all parties present)
- Reference shared history when relevant
- Avoid repeating recent message content

**Pattern-Guided:**
- Apply retrieved rules (good examples, avoid bad patterns)
- Follow situational guidance (immediate acknowledgment after interaction)

### Generation Prompt Structure (Simplified)

```
You are helping [User] maintain their important relationship with [Contact].

CONTEXT:
- Current note: [note text]
- Recent notes: [last 3 notes]
- Relevant history: [vector-retrieved notes]
- Relationship profile: [FORD + summary]
- Relationship metrics: [age, rhythm, closeness level]
- Past messages: [last 5 messages]

RETRIEVED RULES:
[Show 3-5 rules with situation/good/bad/why]

USER'S COMMUNICATION STYLE:
[User's tone examples for this closeness level]

CLOSENESS SCALE: [N]/7 (1=formal, 7=very familiar)

YOUR TASK:
Generate a message that:
1. References actual events/information from notes (evidence-bound)
2. Matches the closeness level and user's style
3. Acknowledges all people present in interactions
4. Follows patterns from retrieved rules
5. Doesn't repeat recent message content

Generate message:
```

**Expected latency**: 2-5 seconds  
**Expected cost**: ~$0.01-0.03 per message (using GPT-4 or Claude Sonnet)

---

## Component 6: User Review & Revision

**Purpose:** Allow user to review and refine generated messages before sending.

### Revision Capabilities

1. **Direct editing** (typing or dictation via Flutter plugin)
2. **LLM-assisted revision** based on user feedback
   - User: "Make it more casual"
   - System: Regenerates with adjusted tone
   - User: "Mention the project deadline"
   - System: Adds specific detail

### Feedback Loop

User revisions provide valuable signals for improving message generation:
- **Accepted messages** (sent as-is): Pattern works well
- **Edited messages** (modified before sending): Capture edit type
  - Tone adjustment (more/less formal)
  - Content addition (user added missing detail)
  - Content removal (user removed incorrect detail)
- **Rejected messages** (deleted): Pattern didn't work

This feedback can inform:
- Rule effectiveness (which rules lead to accepted messages)
- Missing patterns (consistent edits suggest new rule needed)
- Model tuning (systematic issues indicate prompt improvements)

---

## Data Storage & Schema

### Notes Collection

```json
{
  "note_id": "note:2024-11-01#1",
  "contact_id": "c_123",
  "user_id": "u_456",
  "text": "Had lunch with Sarah and John. Sarah's mother had surgery last week...",
  "created_at": "2024-11-01T14:30:00Z",
  "signals": {
    "key_topics": ["family_health", "social_interaction"],
    "named_entities": [...],
    "interaction_type": "in_person_meeting",
    ...
  },
  "embedding": [...],  // Vector for semantic search
  "is_background_info": false
}
```

### Relationship Profiles Collection

```json
{
  "contact_id": "c_123",
  "user_id": "u_456",
  "ford_categories": {
    "family": {
      "value": "...",
      "evidence": {"note:...": "..."},
      "last_updated_at": "..."
    },
    ...
  },
  "relationship_summary": {
    "value": "...",
    "evidence": {...},
    "last_updated_at": "..."
  },
  "relationship_metrics": {
    "relationship_age": {...},
    "interaction_rhythm": {...},
    "closeness_level": {...}
  },
  "updated_at": "2024-11-01T14:30:00Z"
}
```

### Message Rules Collection (Vector Database)

```json
{
  "rule_id": "rule_001",
  "user_id": "u_456",  // User-specific rules or "global" for shared rules
  "situation": "Primary contact and spouse/partner both present...",
  "relationship_context": "Professional relationship with personal rapport...",
  "signals": {
    "key_topics": ["multi_person_meeting"],
    "interaction_type": "in_person",
    "relationship_stage": "established"
  },
  "good_example": "...",
  "bad_example": "...",
  "why": "...",
  "tags": ["multi_person", "acknowledgment"],
  "embedding": [...],  // Vector for semantic search
  "created_at": "2024-11-01T10:00:00Z"
}
```

### Messages Collection

```json
{
  "message_id": "msg_789",
  "contact_id": "c_123",
  "user_id": "u_456",
  "generated_text": "Original generated message...",
  "final_text": "Message after user edits...",
  "was_edited": true,
  "edit_type": "tone_adjustment",
  "sent_at": "2024-11-01T15:00:00Z",
  "rules_used": ["rule_001", "rule_042"],
  "context_notes": ["note:2024-11-01#1", "note:2024-10-01#2"],
  "user_feedback": "more_casual"
}
```

---

## Implementation Priorities

### Phase 1: Core Foundation (V1)
1. ✅ Basic note logging (typing, dictation)
2. ✅ Simple relationship profiles (FORD, summary)
3. ✅ Basic message generation (current note + profile)
4. ✅ User review & revision flow
5. ✅ Closeness scale for tone

### Phase 2: Context Engineering (V1.5)
1. **Signal extraction** from notes
2. **Evidence-based profile updates** with references
3. **Count-based context retrieval** (last N notes)
4. **Basic rule database** with pattern matching
5. **Entity anonymization** for rule queries

### Phase 3: Advanced Retrieval (V2)
1. **Vector search** for historical notes
2. **Relationship metrics** (age, rhythm, closeness level)
3. **Enhanced rule retrieval** with signal filtering
4. **Feedback loop** for rule effectiveness
5. **Message repetition detection**

### Phase 4: Intelligence & Optimization (V2+)
1. Automatic rule generation from user feedback
2. Proactive relationship maintenance suggestions
3. Multi-message coordination (don't send too many at once)
4. Calendar integration for time-sensitive messages
5. Performance optimizations (caching, batch processing)

---

## Key Differences from Predecessor System

| Aspect | Predecessor (Assiist) | VIP |
|--------|----------------------|-----|
| **Primary Output** | Tasks (messages + actions + appointments) | Messages only |
| **Complexity** | High (multi-step task generation, verification) | Simplified (message generation + review) |
| **Context Sources** | Notes, tasks, appointments, availability | Notes, relationship profile, past messages |
| **Retrieval** | Complex (seeds + vector + ranking) | Simpler (seeds + vector, direct assembly) |
| **Validation** | Separate verification step with claim checking | User review (simpler, no separate validation step) |
| **Profile Updates** | Complex (multiple detail types, opportunities) | Focused (FORD + summary + metrics) |
| **Rules/Examples** | Task-oriented (what tasks + how to phrase) | Message-oriented (what to say + tone) |

**Core Similarity:** Both systems use evidence-based profiles, entity anonymization for pattern matching, and count-based retrieval — but VIP simplifies everything for the narrower message generation use case.

---

## Success Metrics

### Quality Metrics
- **Message acceptance rate**: % of generated messages sent without edits (target: >70%)
- **Profile accuracy**: User-reported accuracy of FORD/summary fields (target: >90%)
- **Rule effectiveness**: % of retrieved rules rated helpful by users (target: >80%)
- **Context relevance**: % of retrieved context used in generated message (target: >60%)

### Performance Metrics
- **End-to-end latency**: Note logged → message ready (target: <8 seconds)
- **Signal extraction**: Per-note processing time (target: <500ms)
- **Profile update**: Per-note update time (target: <1 second)
- **Context retrieval**: Gather all context (target: <2 seconds)
- **Message generation**: Generate draft (target: <5 seconds)

### Cost Metrics
- **Signal extraction**: Per-note cost (target: <$0.001)
- **Profile update**: Per-note cost (target: <$0.005)
- **Message generation**: Per-message cost (target: <$0.03)
- **Total per interaction**: Note → message (target: <$0.04)

---

## References

### Predecessor Documentation
- Context Retrieval Overview: `/path/to/predecessor/context_retrieval_overview.md`
- Signal Detection (LLM): `/path/to/predecessor/1_signal_detection_llm.md`
- Situation Summary Builder: `/path/to/predecessor/2_situation_summary_builder.md`
- Retrieval Query Generation: `/path/to/predecessor/1_retrieval_query_generation.md`
- Example Bank Retrieval: `/path/to/predecessor/2_example_bank_retrieval.md`
- Context Sketch Builder: `/path/to/predecessor/3_context_sketch_builder.md`
- Relationship Profile: `/path/to/predecessor/relationship_profile.md`
- Task Generation Spec v2: `/path/to/predecessor/task_generation_spec_v2.md`

### VIP Documentation
- Features Overview: `./features.md`
- Tech Stack: `../agent-os/standards/global/tech-stack.md`

