# Product Roadmap

1. [ ] Account & Authentication Platform — Launch secure account provisioning with passwordless sign-in, optional OAuth providers, session management, and per-user data partitioning so downstream features can assume an authenticated context. `S`

2. [ ] Contact Sync & VIP List Management — Enable users to sync their contacts and manage their VIP list within the app (add/remove contacts), with sync operations that update only the VIP contacts dataset without affecting existing notes or summaries. `S`

3. [ ] Basic Notes System — Implement typed note-taking functionality where users can log interaction history and relationship context, with notes stored individually and associated with specific VIP contacts. `S`

4. [ ] Voice Dictation for Notes — Integrate the Flutter plugin for iOS voice dictation, allowing users to dictate notes hands-free using native APIs through the plugin interface. `M`

5. [ ] Evidence-Based Relationship Profiles — Create contact-level profile system with FORD categories and relationship summary fields that update incrementally from notes, maintaining evidence references (which notes support each piece of information) and timestamps for transparency. `M`

6. [ ] Note Signal Extraction — Build signal extraction system that analyzes notes to identify key topics, named entities (people, organizations, projects), temporal markers, interaction types, and sentiment indicators that inform message generation. `L`

7. [ ] Relationship Metrics Computation — Calculate and maintain relationship metrics (relationship age, interaction rhythm, closeness level) from historical notes, storing evidence references and updating incrementally with each new note as hints for message tone and priority. `M`

8. [ ] Closeness Scale & Tone System — Implement 1-7 closeness scale per contact that users can adjust, defining how this scale impacts message tone, formality, length, and emoji usage during generation. `S`

9. [ ] User Style Examples Collection — Create interface for users to provide message examples across preset categories (Professional, Friendly, Close Friend, Family) that serve as style guides for message generation. `S`

10. [ ] Message Rules Vector Database — Build vector database system for storing and retrieving situation-specific message generation rules using entity anonymization and signal-based pattern matching to find relevant rules for current context. `L`

11. [ ] Context Retrieval System — Implement multi-source context gathering for message generation: recent notes (count-based, last N notes), vector search across historical notes for semantic relevance, relationship profile data, and past messages to avoid repetition. `L`

12. [ ] Birthday Message Generation — Create automated message generation for contact birthdays using retrieved context, relationship profiles, matched rules, and user style examples, with messages stored and ready for user review. `M`

13. [ ] Event-Based Message Generation — Extend message generation to detect relevant events from note content and temporal signals, creating appropriate messages for detected milestones, follow-ups, and "thinking of you" moments based on retrieved rules. `L`

14. [ ] Message Review & Revision Interface — Build notification-driven review screen where users can see generated messages, revise via typing or dictation, with LLM revision based on user feedback before finalizing. `M`

15. [ ] Native Messaging App Integration — Implement deep linking to open native messaging app with message prefilled for the target contact, allowing seamless sending from user's preferred messaging platform. `S`

16. [ ] Push Notification System — Set up push notification infrastructure to alert users when messages are ready for review, with tap-to-open functionality that launches the message review screen. `M`

> Notes
> - Order items by technical dependencies and product architecture
> - Each item represents an end-to-end (frontend + backend) functional and testable feature

