# Product Mission

## Pitch
VIP is a relationship management app that helps busy professionals and individuals maintain meaningful connections by generating thoughtful, personalized messages at the right times. By capturing context from notes and interactions, VIP removes the cognitive load of remembering important dates, preferences, and communication patterns.

## Users

### Primary Customers
- **Busy Professionals**: Individuals who value their network but struggle to maintain consistent, meaningful contact across many relationships
- **Relationship-Focused Individuals**: People who want to be intentional about nurturing their most important personal and professional connections

### User Personas
**Sarah, The Networked Professional** (30-45)
- **Role:** Business owner, consultant, or executive with extensive professional network
- **Context:** Manages 20-50 VIP relationships (clients, partners, mentors) alongside family and close friends
- **Pain Points:** Forgets birthdays, loses context between meetings, feels guilty about neglecting relationships, struggles to maintain personal touch at scale
- **Goals:** Stay top-of-mind with key contacts, demonstrate genuine care, maintain relationships effortlessly without sacrificing authenticity

**David, The Intentional Friend** (25-40)
- **Role:** Professional who prioritizes quality relationships over quantity
- **Context:** Has 10-20 very close relationships (family, best friends, key colleagues) that matter deeply
- **Pain Points:** Forgets important details from conversations, misses opportunities to follow up on things that matter to loved ones, wants to be more thoughtful
- **Goals:** Be the friend/family member who remembers and cares about the details, deepen existing relationships through consistent, meaningful contact

## The Problem

### Relationship Maintenance Doesn't Scale With Good Intentions
People genuinely want to maintain meaningful relationships, but the cognitive overhead of tracking context across dozens of connections is overwhelming. Important birthdays slip by, conversation details are forgotten, and follow-ups fall through the cracks. Traditional contact management focuses on storing information, not generating action. Calendar reminders are impersonal and don't help craft thoughtful messages. The result: relationships weaken not from lack of care, but from lack of cognitive capacity.

**Our Solution:** VIP captures relationship context through simple note-taking (typed or dictated), then generates personalized messages that reflect the appropriate tone and reference real interactions. Users review, refine, and send messages knowing they're grounded in actual relationship history.

## Differentiators

### Context-Driven Message Generation
Unlike generic reminder apps or CRMs, VIP uses your actual interaction history and notes to generate messages that sound like you and reference real events. The system extracts signals from notes (key topics, entities, temporal markers, sentiment) to determine what messages to create, when to send them, and how to phrase them appropriately.

### Evidence-Based Relationship Intelligence
Unlike AI tools that hallucinate or make assumptions, VIP maintains evidence-bound profiles where every piece of information traces back to specific notes. Profile updates are incremental and conservative, ensuring the system only persists information explicitly supported by your documented interactions. This creates trustworthy relationship context that users can rely on.

### Tone Calibration Through Closeness Levels
Unlike one-size-fits-all messaging tools, VIP adapts tone, word choice, length, and formality to match the closeness of each relationship. Users provide style examples for different relationship types (professional, friendly, close friend, family), and the system generates messages that match both the relationship level and the user's personal communication style.

## Key Features

### Core Features
- **Note Capture System:** Log relationship context through typing or voice dictation (iOS-first Flutter plugin), capturing interaction history, important details, and background information that feeds intelligent message generation
- **Intelligent Message Generation:** Automatically create personalized messages for birthdays and events detected from notes, using retrieved context, relationship profiles, and dynamically matched rules to ensure messages are timely, relevant, and appropriately toned
- **Message Review & Revision:** Receive push notifications when messages are ready, review and revise using typing or dictation, then send through native messaging app with message prefilled for seamless delivery
- **VIP Contact Management:** Sync contacts and curate your VIP list within the app, adding or removing contacts as relationships evolve without affecting existing notes or relationship history

### Relationship Intelligence Features
- **Evidence-Based Profiles:** Maintain structured relationship context (FORD categories, relationship summary, closeness metrics) where each piece of information is grounded in specific notes with timestamps and evidence references
- **Signal Extraction & Analysis:** Automatically identify key topics, named entities, temporal markers, interaction types, and sentiment from notes to improve message generation quality and timing accuracy
- **Context Retrieval System:** Intelligently gather relevant context from recent notes, historical semantic search, relationship profiles, and past messages to ensure generated messages reference real events and avoid repetition

### Personalization Features
- **Closeness Scale & Tone Calibration:** Adjust relationship closeness levels (1-7 scale) per contact to control message tone, formality, length, and emoji usage, ensuring communications match the nature of each relationship
- **Personal Style Examples:** Provide message examples for different relationship types (professional, friendly, close friend, family) that guide the system to match your unique communication voice and style preferences
- **Dynamic Rule Matching:** Leverage a vector database of situation-specific message rules that retrieve based on note context, ensuring messages follow appropriate patterns (e.g., acknowledging multiple people present, following up on specific events)

