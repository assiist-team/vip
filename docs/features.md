VIP App — Product Context & Vision

VIP is an app designed for people who want to intentionally nurture their most important relationships—family, friends, clients, colleagues, and other high-value contacts. The app removes the cognitive load of remembering important dates, context, and communication patterns by generating thoughtful, personalized messages at the right times. Users capture notes—typed or dictated via an in-house Flutter plug-in—about interactions, background details, goals, and preferences. The system uses these notes to generate messages (e.g., birthday wishes, follow-ups, "thinking of you" nudges) that reflect the tone and closeness appropriate for each relationship. Users can review, revise, and send messages via their native messaging app. VIP is iOS-first, with future expansion to other platforms through Flutter. It is aimed at busy professionals and individuals who want to maintain meaningful personal and professional relationships consistently.

**Note:** For technical implementation details about the context engineering system (signal extraction, evidence-based profiles, retrieval strategies, rule matching), see [`context-engineering.md`](./context-engineering.md).

⸻

VIP App — Feature Breakdown (V1 + V2 Candidates)

1. Contact Sync & VIP List Management
	•	Users can sync contacts and manage their VIP list entirely within the app:
	•	Add contacts to VIP
	•	Remove contacts from VIP
	•	Sync updates only the VIP contacts dataset (does not affect notes or summaries).
	•	The interface is designed to be intuitive, letting users adjust their VIP list at any time.

⸻

2. Notes System (Typed or Dictated)
	•	Users can log notes using:
	•	Typing
	•	Dictation via the in-house Flutter plugin (currently iOS-only)
	•	Notes are stored individually and used to:
	•	Capture relationship history, context, and important details
	•	Feed content into structured summary fields (e.g., FORD, relationship summary) without saving notes directly to these fields
	•	Notes may optionally be designated as background information about the relationship; this is context-dependent and uses the same note object.

2.1 Note Analysis & Signal Extraction
	•	When a note is logged, the system extracts structured signals to improve message generation quality:
	•	Key topics/themes mentioned in the note
	•	Named entities (people present, organizations, projects/events referenced)
	•	Temporal signals (dates, times, relative time expressions)
	•	Interaction type (meeting, call, text exchange, etc.)
	•	Sentiment/urgency indicators
	•	These signals help determine:
	•	What messages to generate (based on detected events/milestones)
	•	When to send messages (based on temporal context)
	•	How to phrase messages (acknowledging multiple people, referencing specific events)
	•	Signals are hints for the generation system, not stored as separate user-facing data

2.2 V2 Consideration
	•	Periodically prompt the user to provide background information if none exists for a contact.

⸻

3. Relationship Profiles (Contact-Level Metadata)

Each VIP contact has a profile including:
	•	FORD categories for the contact: Family, Occupation, Recreation, Dreams
	•	Relationship Summary: user's personal connection and history with the contact

3.1 Evidence-Based Profile Updates
	•	Profile information is evidence-bound: each piece of information traces back to specific notes
	•	Profile updates happen incrementally as notes are logged
	•	Each profile field maintains:
	•	The current value (text describing the person)
	•	Evidence references (which notes support this information)
	•	Last updated timestamp
	•	This approach ensures:
	•	Profile information remains grounded in actual interactions
	•	Outdated information can be identified and refreshed
	•	Users can trust the accuracy of relationship context
	•	Conservative updates: only persist information explicitly supported by notes

3.2 Relationship Metrics (Hints for Tone/Priority)
	•	The system computes relationship metrics from historical notes and interactions:
	•	Relationship age (years/months known) — temporal depth for shared context
	•	Interaction rhythm (days since last contact + typical cadence) — engagement recency and pattern
	•	Closeness level (references the 1–7 scale from Section 5, user-adjustable) — emotional intimacy and formality
	•	These metrics are hints used for message generation (tone, urgency, level of detail)
	•	Each metric includes evidence references to support its value
	•	Metrics update incrementally with each note
	•	Note: Communication style (formal, friendly, casual, familial) is derived from closeness level during message generation rather than stored as a separate metric

⸻

4. Message Generation System
	•	Messages are generated:
	•	On birthdays (from contact data)
	•	In proximity to relevant events implicitly recognized from notes (LLM interprets context)

4.1 Message Rules Vector Database
	•	A dynamic vector database stores message generation rules that can be retrieved based on context.
	•	Rules are situation-specific guidelines that influence message generation (e.g., "If the note references an interaction that just happened, always create a message acknowledging that interaction").
	•	Before message generation, the system performs a vector search step:
	•	The system builds a query using entity anonymization:
	•	Specific names/organizations/projects are replaced with type placeholders ([CONTACT], [ORG], [PROJECT])
	•	Combined with extracted signals (key topics, interaction type, sentiment)
	•	This enables pattern-based matching on functional similarity rather than surface features
	•	Example: "Had lunch with [CONTACT] and [CONTACT]'s spouse" matches rules about multi-person interactions
	•	The system retrieves relevant rules from the vector database that match the current situation pattern.
	•	Retrieved rules are incorporated into the main prompt used for message generation.
	•	Multiple messages can be generated from a single note, depending on the note content and retrieved rules, potentially for different points in time (present or future).

4.1.1 Rule Structure & Pattern Matching
	•	Each rule includes:
	•	Situation pattern (generic/abstract description)
	•	Relationship context (type of relationship, familiarity level)
	•	Good/bad examples (demonstrating the pattern)
	•	Why explanation (reasoning behind the rule)
	•	Signal tags (for filtering: key_topics, interaction_type, relationship_stage)
	•	Rules are written generically (no specific names) so they match on functional patterns
	•	Example rule: "Multi-person meeting pattern"
	•	Situation: "Primary contact and spouse/partner both present at interaction"
	•	Good: "Good seeing you both yesterday..."
	•	Bad: "Great meeting you today..." (doesn't acknowledge both people)
	•	Why: "Acknowledge all parties present in multi-person interactions"
	•	This approach allows a single rule to guide both what to say and how to say it

4.2 Context Retrieval for Message Generation
	•	When generating a message, the system gathers relevant context from multiple sources:
	•	Recent notes: Last N notes for the contact (count-based, not time-based)
	•	Example: Always retrieve last 3 notes regardless of when they were logged
	•	Rationale: "Last 3 notes" is more meaningful than "last 30 days" for varying interaction frequencies
	•	Vector retrieval: Semantic search across all historical notes for relevant context
	•	Uses extracted signals (key topics, entities, dates) to build search queries
	•	Retrieves older relevant notes that might inform the current message
	•	Example: If note mentions "their mother's surgery," retrieve past notes about their mother's health
	•	Relationship profile: FORD categories, relationship summary, relationship metrics
	•	Used as hints for tone and context, not as citeable facts
	•	Past messages: Recent messages sent to avoid repetition
	•	All retrieved context is used to ensure:
	•	Messages reference real events and information
	•	Messages don't repeat recent content
	•	Tone matches relationship closeness and communication style
	•	Important context isn't missed

4.3 Notification & Revision Flow
	•	User receives a push notification when a message is ready.
	•	Tap opens the message review screen.
	•	User can revise the message:
	•	Typing
	•	Dictation (via Flutter plugin)
	•	LLM revises the message in place based on user feedback.
	•	Once ready, user taps a button to open their native messaging app with the message prefilled for the contact.

⸻

5. Familiarity / Closeness Scale (Tone System)
	•	Uses a numeric scale (e.g., 1–5 or 1–7) to represent closeness/formality.
	•	Impacts:
	•	Tone
	•	Word choice
	•	Length of message
	•	Emoji usage or colloquial language
	•	Users can adjust the scale per contact at any time.

5.1 User-Provided Tone Examples
	•	Preset categories (e.g., Professional, Friendly, Close Friend, Family) are provided.
	•	Users supply examples fitting each preset category, guiding the LLM on their personal messaging style.
	•	The LLM uses these examples as a style guide when generating messages.

⸻

6. App Platform Strategy
	•	iOS-first using Flutter
	•	Dictation is handled entirely through the in-house Flutter plugin (native APIs are used under the hood, but the app uses the plugin interface)
	•	Android and other platforms are planned for a future phase.

⸻

V2 Features / Considerations
	•	Periodic prompts to fill in missing relationship background info
	•	Configurable intervals for “check-in” messages

