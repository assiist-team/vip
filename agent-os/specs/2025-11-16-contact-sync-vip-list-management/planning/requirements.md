# Spec Requirements: Contact Sync & VIP List Management

## Initial Description
Contact Sync & VIP List Management — Enable users to sync their contacts and manage their VIP list within the app (add/remove contacts), with sync operations that update only the VIP contacts dataset without affecting existing notes or summaries.

## Requirements Discussion

### First Round Questions

**Q1:** Contact Permission Flow - I assume we'll request iOS contact permissions on first launch or when the user first attempts to access the VIP list management screen, following iOS best practices with a clear explanation of why we need access. Is that correct, or should we request permissions at a different time?
**Answer:** Request permissions on first launch or when first accessing VIP list management screen.

**Q2:** Contact Information to Sync - I'm thinking we should sync basic contact info (name, phone number, email, profile photo if available) for each VIP contact. Should we also include additional fields like birthday, company, job title, or just keep it minimal?
**Answer:** Sync name, phone number, email, profile photo, and birthday. Cap it at these fields.

**Q3:** VIP List Management UI - I assume the VIP management screen will show all device contacts in a searchable/scrollable list with a toggle or checkbox to mark contacts as VIPs, similar to iOS Favorites. Should we group contacts alphabetically, show a separate "Current VIPs" section at the top, or use a different layout approach?
**Answer:** Have separate sections - "My VIPs" at the top showing currently selected VIPs, and "All Contacts" below. Both sections should group contacts alphabetically. Include a search bar at the top to quickly find contacts regardless of section.

**Q4:** Sync Operation Behavior - You mentioned "sync operations that update only the VIP contacts dataset without affecting existing notes or summaries." I'm assuming this means: Initial sync imports all selected VIP contacts; subsequent syncs update VIP contact info if changed on device but preserve all relationship notes, profiles, and message history; when a contact is removed from VIP list, keep all their historical data but mark them as inactive/non-VIP. Is that the intended behavior, or should removing a VIP also archive/delete their data?
**Answer:** Assumptions are correct. Initial sync imports selected VIPs, subsequent syncs update contact info while preserving notes/profiles/history, and removing from VIP list keeps historical data but marks as inactive.

**Q5:** Sync Trigger & Frequency - Should the sync happen automatically in the background (e.g., on app launch or periodically), or should users manually trigger syncs via a "Refresh" button? Or both?
**Answer:** Automatic background syncs, but also include a refresh button (especially useful during development).

**Q6:** Contact Matching & Updates - When a user edits a VIP contact's info on their device (like changing their phone number), I assume we should detect and update that contact in our database. Should we notify the user about these changes, or handle them silently?
**Answer:** Detect and update contact info in database silently. This applies to phone number, email, name, profile photo, and birthday changes.

**Q7:** Multiple Contacts with Same Name - How should we handle duplicate names? I'm thinking we should show additional context (phone number or email) in the UI to distinguish between contacts. Sound good?
**Answer:** Use phone number in the UI to distinguish contacts with the same name.

**Q8:** Scope Boundaries - Is there anything we should explicitly NOT include in this feature? For example, should we skip contact merging/de-duplication, skip syncing contact groups, or avoid any other edge cases for now?
**Answer:** Keep it simple for iOS-first implementation. Deduplication is important - don't want multiple entries for the same contact. Phone number is the "field of truth" for identifying unique contacts. Design for eventual cross-platform support but implement iOS first.

### Existing Code to Reference

No similar existing features identified for reference. This is the first feature implementation for the VIP app.

### Follow-up Questions

**Follow-up 1:** You mentioned separate sections with current VIPs. Just to confirm the layout - should it be: Section 1 (top) "My VIPs" showing only contacts already marked as VIPs (alphabetically grouped), and Section 2 (below) "All Contacts" showing all device contacts not yet marked as VIPs (alphabetically grouped). Is that the right structure?
**Answer:** Yes, that's correct. Top section shows "My VIPs" with contacts already marked as VIPs. When a new contact is marked as VIP, they should move up into that section. Search bar at the top allows users to quickly find contacts and move them between sections (from Regular to VIPs or VIPs to Regular).

**Follow-up 2:** For contact deduplication, are you concerned about: Scenario A - the same contact appearing multiple times in the device's contact list (e.g., "John Smith" with different phone numbers), or Scenario B - preventing users from accidentally adding the same contact as a VIP multiple times through our app? For Scenario A, we could match contacts by phone number or email. For Scenario B, this would be handled automatically by database constraints. Which scenario concerns you?
**Answer:** Scenario B should be handled automatically through the section movement design (when a contact is marked as VIP, they move sections, so they can't be added twice). Phone number is the field of truth for identifying unique contacts.

**Follow-up 3:** You mentioned eventual cross-platform support. For this first iOS implementation, should we design the database schema and sync logic to be platform-agnostic (e.g., storing a contact source identifier), or is it fine to keep it iOS-specific for now and refactor later when Android is added?
**Answer:** Design the database schema and sync logic to be platform-agnostic from the start.

## Visual Assets

### Files Provided:
No visual assets provided.

### Visual Insights:
N/A - No visuals provided.

## Requirements Summary

### Functional Requirements
- **Contact Sync:** Sync contacts from iOS device including name, phone number, email, profile photo, and birthday
- **VIP List Management:** Allow users to mark/unmark contacts as VIPs through an intuitive UI with section-based organization
- **Two-Section UI:** Display "My VIPs" section at top (alphabetically grouped) and "All Contacts" section below (alphabetically grouped)
- **Search Functionality:** Provide search bar to quickly find any contact regardless of which section they're in
- **Contact Movement:** When a contact is marked as VIP, move them from "All Contacts" to "My VIPs" section automatically
- **Contact Updates:** Automatically detect and sync changes to contact info (name, phone, email, photo, birthday) from device
- **Permission Flow:** Request iOS contact permissions on first launch or first access to VIP management screen
- **Background Sync:** Automatically sync contacts in background (on app launch or periodically)
- **Manual Refresh:** Provide refresh button for manual sync triggering
- **Contact Deduplication:** Use phone number as unique identifier to prevent duplicate contact entries
- **Name Disambiguation:** Display phone number alongside name in UI when multiple contacts share the same name
- **Data Preservation:** When contact is removed from VIP list, preserve all historical notes, profiles, and message history (mark as inactive, don't delete)

### Reusability Opportunities
- No existing components identified (first feature implementation)
- Future features will likely reuse:
  - Contact list UI patterns established here
  - Contact data models and sync logic
  - Permission request flows

### Scope Boundaries
**In Scope:**
- iOS contact syncing (name, phone, email, photo, birthday)
- VIP list management (add/remove contacts as VIPs)
- Two-section UI with alphabetical grouping and search
- Automatic background sync and manual refresh option
- Contact update detection and silent syncing
- Phone number-based deduplication
- Platform-agnostic database schema design
- Data preservation when contacts removed from VIP list

**Out of Scope:**
- Contact groups syncing
- Complex contact merging/deduplication for device-level duplicates (Scenario A)
- User notifications about contact info changes (handle silently)
- Android implementation (future phase)
- Contact editing within the app (changes made on device only)
- Syncing additional contact fields beyond the specified five (name, phone, email, photo, birthday)
- Contact creation within the app

### Technical Considerations
- **Platform-Agnostic Design:** Database schema and sync logic must be designed to support future Android implementation
- **Phone Number as Primary Key:** Use phone number as the unique identifier/"field of truth" for contacts
- **Data Integrity:** Sync operations must never affect existing notes, summaries, or relationship profiles
- **Flutter Contacts Plugin:** Use Flutter contacts plugin for iOS contact access
- **Permission Handling:** Follow iOS best practices for contact permission requests with clear explanations
- **State Management:** Use Riverpod for managing VIP list state and contact sync status
- **Supabase Database:** Store synced contact data in Supabase PostgreSQL with proper RLS policies
- **Inactive Status:** Implement contact status field to mark contacts as inactive when removed from VIP list (preserves historical data)
- **Search Implementation:** Implement real-time search filtering across both UI sections

