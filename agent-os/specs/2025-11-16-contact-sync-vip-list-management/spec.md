# Specification: Contact Sync & VIP List Management

**Created:** 2025-11-16  
**Status:** Draft  
**Feature Type:** Core Feature  

---

## 1. Overview

### 1.1 Feature Description
Enable users to sync their contacts from iOS devices and manage their VIP list within the app. Users can add or remove contacts as VIPs through an intuitive interface with automatic background sync and manual refresh capabilities. Sync operations update contact information without affecting existing notes, summaries, or relationship profiles.

### 1.2 Goals & Objectives
- **Primary Goal:** Provide seamless contact syncing and VIP management to enable relationship tracking within the VIP app
- **User Experience Goal:** Create an intuitive, friction-free interface for managing VIP contacts with clear visual organization
- **Technical Goal:** Build platform-agnostic sync infrastructure that preserves data integrity and supports future Android implementation

### 1.3 Success Metrics
- **Adoption:** 90%+ of users complete initial contact sync within first session
- **Engagement:** Users maintain an average of 10-50 VIP contacts
- **Reliability:** 99%+ success rate for contact sync operations
- **Performance:** Initial sync completes within 5 seconds for up to 100 contacts
- **Data Integrity:** Zero data loss incidents (notes/profiles preserved across all sync operations)

### 1.4 Dependencies & Preconditions
- **Auth/Login Prerequisite:** Contact sync only begins once the `2025-11-16-auth-login-feature` flow has produced a verified `supabase.auth.currentSession` and the onboarding tutorial (`profiles.onboarding_completed_at`) is complete. Unauthenticated or unverified users remain in the auth stack and never hit this surface.
- **Session Continuity:** The Contact Sync service must subscribe to `supabase.auth.onAuthStateChange` so it can pause sync, clear cached contact state, and route users back through the auth gate if the session becomes invalid.
- **Profiles Table Alignment:** Contact records reference `auth.users.id`; logout/account deletion (handled by the auth spec) must cascade by design, so contact code cannot assume records persist after those events.

---

## 2. User Experience

### 2.1 User Flows

#### Initial Setup Flow
1. User completes the Auth/Login flow (email/password or Google), verifies email, and finishes onboarding so they enter the authenticated app shell.
2. App verifies an active Supabase session is present (rehydrated via `auth.login` secure storage work); if absent, the user is routed back to the auth stack.
3. User navigates to the VIP management screen for the first time.
4. App requests iOS contact permissions with clear explanation:
   - **Permission Prompt:** "VIP needs access to your contacts to help you manage relationships with the people who matter most."
5. User grants permission
6. App records permission status locally and in the profile metadata for analytics (non-blocking).
7. App displays VIP management screen with empty "My VIPs" section
8. User searches or scrolls through "All Contacts" section
9. User taps contacts to mark them as VIPs
10. Selected contacts move to "My VIPs" section
11. Initial sync completes, VIP contacts stored in database

#### Subsequent Access Flow
1. User navigates to VIP management screen from within an authenticated session.
2. App re-validates session freshness (leveraging the auth feature’s session rehydration + biometric gate if enabled); if expired it shows the login screen instead of the contact UI.
3. App performs automatic background sync (silent, no loading indicator unless taking >2 seconds)
3. UI displays:
   - **Section 1 (Top):** "My VIPs" - alphabetically grouped contacts already marked as VIPs
   - **Section 2 (Below):** "All Contacts" - alphabetically grouped contacts not yet marked as VIPs
4. User can:
   - Search for contacts using search bar at top
   - Mark new contacts as VIPs (moves to "My VIPs" section)
   - Remove VIP status (moves to "All Contacts" section)
   - Pull down to manually refresh sync
5. Changes persist immediately to database

#### Contact Update Flow (Background)
1. User updates contact info on iOS device (name, phone, email, photo, birthday)
2. On next app launch or background sync:
   - App detects changes via contact identifier matching
   - Updates contact info in database silently
   - All notes, profiles, and history preserved
   - No user notification shown

#### VIP Removal Flow
1. User taps to remove VIP status from contact
2. Contact moves from "My VIPs" to "All Contacts" section
3. Database marks contact as inactive (status field)
4. All historical data preserved (notes, profiles, messages)
5. Contact can be re-added as VIP later, restoring active status

### 2.2 Interface Design

#### VIP Management Screen Layout
```
┌─────────────────────────────────────┐
│  [Search bar with icon]             │
├─────────────────────────────────────┤
│                                     │
│  MY VIPs (12)                       │
│  ─────────────                      │
│  A                                  │
│  📷 Alice Johnson                   │
│      (555) 123-4567                 │
│  📷 Andrew Smith                    │
│      (555) 234-5678                 │
│                                     │
│  B                                  │
│  📷 Bob Williams                    │
│      (555) 345-6789                 │
│  ...                                │
│                                     │
├─────────────────────────────────────┤
│                                     │
│  ALL CONTACTS (248)                 │
│  ─────────────                      │
│  C                                  │
│  📷 Carol Davis                     │
│      (555) 456-7890                 │
│  ...                                │
│                                     │
└─────────────────────────────────────┘
```

**Key UI Elements:**
- **Search Bar:** Fixed at top, searches across all contacts (both sections)
- **Section Headers:** Clear labels with counts ("MY VIPs (12)", "ALL CONTACTS (248)")
- **Alphabetical Groups:** Letter headers for each alphabetical section (A, B, C...)
- **Contact Cards:** 
  - Profile photo (if available) or placeholder icon
  - Name prominently displayed
  - Phone number shown below name (for disambiguation)
  - Tap to toggle VIP status
  - Visual indicator (checkmark or star icon) for VIP contacts
- **Pull-to-Refresh:** Swipe down gesture triggers manual sync with loading indicator
- **Empty States:**
  - "My VIPs" empty: "No VIPs yet. Tap contacts below to add them."
  - Search no results: "No contacts found for '[search query]'"

#### Contact Card States
1. **Standard State:** Contact with photo/icon, name, phone number
2. **VIP State:** Same as standard + visual indicator (star icon or checkmark)
3. **Loading State:** Subtle spinner or shimmer during sync operations
4. **Duplicate Name State:** Shows phone number more prominently when multiple contacts share name

### 2.3 Interaction Patterns
- **Single Tap:** Toggle VIP status (add to/remove from VIP list)
- **Pull Down:** Manual refresh/sync
- **Type in Search:** Real-time filtering across all contacts
- **Scroll:** Navigate through alphabetically grouped contacts in both sections

### 2.4 Error Handling & Edge Cases

#### Permission Denied
- **Scenario:** User denies contact access
- **Handling:** Show friendly message: "VIP needs contact access to help you manage relationships. You can enable this in Settings > VIP > Contacts."
- **UI:** Display settings deep-link button

#### Sync Failure
- **Scenario:** Network error or sync operation fails
- **Handling:** Show non-intrusive error banner: "Couldn't sync contacts. Pull down to try again."
- **Recovery:** Retry button, automatic retry on next app launch

#### Duplicate Names
- **Scenario:** Multiple contacts named "John Smith"
- **Handling:** Display phone number prominently below name for all contacts with same name
- **Example:** 
  - John Smith (555) 123-4567
  - John Smith (555) 987-6543

#### Contact Without Phone Number
- **Scenario:** Contact has email but no phone number
- **Handling:** Use email as identifier fallback, display email in UI where phone would appear
- **Scope Note:** Primary implementation assumes phone number; email-only support deferred to future phase

#### Large Contact Lists
- **Scenario:** User has 500+ contacts
- **Handling:** 
  - Implement virtualized scrolling for performance
  - Show loading indicator only if initial render takes >2 seconds
  - Cache contact list for faster subsequent loads

---

## 3. Technical Architecture

### 3.1 System Components

#### Frontend (Flutter)
**Components:**
1. **VipManagementScreen** - Main screen widget
2. **ContactListSection** - Reusable section widget (for "My VIPs" and "All Contacts")
3. **ContactCard** - Individual contact display widget
4. **SearchBar** - Contact search input widget
5. **ContactSyncService** - Service layer for contact operations
6. **ContactStateNotifier** - Riverpod state management for contact list
7. **PermissionHandler** - iOS contact permission management

**State Management (Riverpod):**
- `contactListProvider` - StateNotifierProvider for contact list state
- `vipContactsProvider` - Computed provider for VIP-only contacts
- `allContactsProvider` - Computed provider for non-VIP contacts
- `contactSyncStatusProvider` - StateProvider for sync loading state
- `searchQueryProvider` - StateProvider for search text

#### Backend (Supabase)
**Database Tables:**
1. **`contacts`** - Stores synced contact data
2. **`contact_sync_logs`** - Audit trail for sync operations (optional)

**Edge Functions:**
- Not required for initial implementation (use direct Supabase client queries)
- Future: Consider Edge Function for batch contact processing if needed

### 3.2 Data Models

#### Database Schema

##### `contacts` Table
```sql
CREATE TABLE contacts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  
  -- Contact Identifiers (Platform-Agnostic)
  phone_number TEXT NOT NULL, -- Primary unique identifier
  platform_contact_id TEXT NOT NULL, -- iOS contact identifier
  platform_source TEXT NOT NULL DEFAULT 'ios', -- 'ios', 'android', 'manual' (future)
  
  -- Contact Information
  full_name TEXT NOT NULL,
  email TEXT,
  profile_photo_url TEXT,
  birthday DATE,
  
  -- VIP Management
  is_vip BOOLEAN NOT NULL DEFAULT false,
  status TEXT NOT NULL DEFAULT 'active', -- 'active', 'inactive'
  
  -- Metadata
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  last_synced_at TIMESTAMPTZ,
  
  -- Constraints
  UNIQUE(user_id, phone_number), -- Prevent duplicates per user
  CHECK (status IN ('active', 'inactive')),
  CHECK (platform_source IN ('ios', 'android', 'manual'))
);

-- Indexes
CREATE INDEX idx_contacts_user_id ON contacts(user_id);
CREATE INDEX idx_contacts_phone_number ON contacts(phone_number);
CREATE INDEX idx_contacts_is_vip ON contacts(user_id, is_vip);
CREATE INDEX idx_contacts_status ON contacts(status);

-- RLS Policies
ALTER TABLE contacts ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can view own contacts"
  ON contacts FOR SELECT
  USING (auth.uid() = user_id);

CREATE POLICY "Users can insert own contacts"
  ON contacts FOR INSERT
  WITH CHECK (auth.uid() = user_id);

CREATE POLICY "Users can update own contacts"
  ON contacts FOR UPDATE
  USING (auth.uid() = user_id);

CREATE POLICY "Users can delete own contacts"
  ON contacts FOR DELETE
  USING (auth.uid() = user_id);

-- Trigger for updated_at
CREATE TRIGGER set_contacts_updated_at
  BEFORE UPDATE ON contacts
  FOR EACH ROW
  EXECUTE FUNCTION update_updated_at_column();
```

#### Flutter Data Models

```dart
// lib/models/contact.dart

import 'package:freezed_annotation/freezed_annotation.dart';

part 'contact.freezed.dart';
part 'contact.g.dart';

@freezed
class Contact with _$Contact {
  const factory Contact({
    required String id,
    required String userId,
    required String phoneNumber,
    required String platformContactId,
    required String platformSource,
    required String fullName,
    String? email,
    String? profilePhotoUrl,
    DateTime? birthday,
    required bool isVip,
    required String status,
    required DateTime createdAt,
    required DateTime updatedAt,
    DateTime? lastSyncedAt,
  }) = _Contact;

  factory Contact.fromJson(Map<String, dynamic> json) =>
      _$ContactFromJson(json);
}

// Contact status enum
enum ContactStatus {
  active,
  inactive;

  String get value => name;
}

// Platform source enum
enum PlatformSource {
  ios,
  android,
  manual;

  String get value => name;
}
```

### 3.3 API Integration

#### Supabase Queries (via PostgREST)

**Fetch All Contacts for User:**
```dart
final contacts = await supabase
    .from('contacts')
    .select()
    .eq('user_id', userId)
    .eq('status', 'active')
    .order('full_name');
```

**Fetch VIP Contacts Only:**
```dart
final vipContacts = await supabase
    .from('contacts')
    .select()
    .eq('user_id', userId)
    .eq('is_vip', true)
    .eq('status', 'active')
    .order('full_name');
```

**Insert New Contact:**
```dart
await supabase.from('contacts').insert({
  'user_id': userId,
  'phone_number': contact.phoneNumber,
  'platform_contact_id': contact.platformContactId,
  'platform_source': 'ios',
  'full_name': contact.fullName,
  'email': contact.email,
  'profile_photo_url': contact.profilePhotoUrl,
  'birthday': contact.birthday?.toIso8601String(),
  'is_vip': contact.isVip,
  'status': 'active',
  'last_synced_at': DateTime.now().toIso8601String(),
});
```

**Update Contact:**
```dart
await supabase.from('contacts').update({
  'full_name': updatedContact.fullName,
  'email': updatedContact.email,
  'profile_photo_url': updatedContact.profilePhotoUrl,
  'birthday': updatedContact.birthday?.toIso8601String(),
  'last_synced_at': DateTime.now().toIso8601String(),
}).eq('id', contactId);
```

**Toggle VIP Status:**
```dart
await supabase.from('contacts').update({
  'is_vip': !currentIsVip,
  'updated_at': DateTime.now().toIso8601String(),
}).eq('id', contactId);
```

**Mark Contact as Inactive:**
```dart
await supabase.from('contacts').update({
  'status': 'inactive',
  'is_vip': false,
  'updated_at': DateTime.now().toIso8601String(),
}).eq('id', contactId);
```

### 3.4 Contact Sync Logic

#### iOS Contact Access (Flutter Plugin)
**Plugin:** `flutter_contacts` (https://pub.dev/packages/flutter_contacts)

**Permission Request:**
```dart
import 'package:flutter_contacts/flutter_contacts.dart';

Future<bool> requestContactPermission() async {
  if (!await FlutterContacts.requestPermission()) {
    return false;
  }
  return true;
}
```

**Fetch All Device Contacts:**
```dart
Future<List<FlutterContact>> fetchDeviceContacts() async {
  if (!await FlutterContacts.requestPermission()) {
    throw ContactPermissionDeniedException();
  }
  
  return await FlutterContacts.getContacts(
    withProperties: true, // Include phone, email, photo
    withPhoto: true,
  );
}
```

#### Sync Algorithm

**Initial Sync (First Time):**
```dart
Future<void> performInitialSync() async {
  // 1. Fetch all device contacts
  final deviceContacts = await fetchDeviceContacts();
  
  // 2. Fetch existing database contacts
  final dbContacts = await fetchDatabaseContacts();
  
  // 3. Create map for quick lookup
  final dbContactsByPhone = {
    for (var c in dbContacts) c.phoneNumber: c
  };
  
  // 4. Process each device contact
  for (final deviceContact in deviceContacts) {
    final phoneNumber = extractPrimaryPhoneNumber(deviceContact);
    if (phoneNumber == null) continue; // Skip contacts without phone
    
    if (!dbContactsByPhone.containsKey(phoneNumber)) {
      // New contact - insert
      await insertContact(
        phoneNumber: phoneNumber,
        platformContactId: deviceContact.id,
        fullName: deviceContact.displayName,
        email: extractPrimaryEmail(deviceContact),
        profilePhotoUrl: await uploadProfilePhoto(deviceContact.photo),
        birthday: extractBirthday(deviceContact),
        isVip: false, // Not VIP by default
      );
    }
  }
}
```

**Subsequent Sync (Update Detection):**
```dart
Future<void> performIncrementalSync() async {
  // 1. Fetch all device contacts
  final deviceContacts = await fetchDeviceContacts();
  
  // 2. Fetch existing database contacts (active VIPs + recently synced)
  final dbContacts = await fetchDatabaseContacts();
  
  // 3. Create maps for efficient lookup
  final deviceContactsByPhone = {
    for (var c in deviceContacts) 
      extractPrimaryPhoneNumber(c): c
  };
  
  final dbContactsByPhone = {
    for (var c in dbContacts) c.phoneNumber: c
  };
  
  // 4. Update existing contacts with changes
  for (final dbContact in dbContacts) {
    final deviceContact = deviceContactsByPhone[dbContact.phoneNumber];
    
    if (deviceContact == null) {
      // Contact deleted from device - keep in DB but mark inactive
      // Only if they're not a VIP (preserve VIP contacts)
      if (!dbContact.isVip) {
        await markContactInactive(dbContact.id);
      }
      continue;
    }
    
    // Check if any fields changed
    if (hasContactChanged(dbContact, deviceContact)) {
      await updateContact(
        contactId: dbContact.id,
        fullName: deviceContact.displayName,
        email: extractPrimaryEmail(deviceContact),
        profilePhotoUrl: await uploadProfilePhotoIfChanged(
          dbContact.profilePhotoUrl,
          deviceContact.photo,
        ),
        birthday: extractBirthday(deviceContact),
      );
    }
  }
  
  // 5. Add new contacts that appeared since last sync
  for (final deviceContact in deviceContacts) {
    final phoneNumber = extractPrimaryPhoneNumber(deviceContact);
    if (phoneNumber == null) continue;
    
    if (!dbContactsByPhone.containsKey(phoneNumber)) {
      await insertContact(
        phoneNumber: phoneNumber,
        platformContactId: deviceContact.id,
        fullName: deviceContact.displayName,
        email: extractPrimaryEmail(deviceContact),
        profilePhotoUrl: await uploadProfilePhoto(deviceContact.photo),
        birthday: extractBirthday(deviceContact),
        isVip: false,
      );
    }
  }
}
```

**Helper Functions:**
```dart
String? extractPrimaryPhoneNumber(FlutterContact contact) {
  if (contact.phones.isEmpty) return null;
  // Normalize phone number (remove formatting)
  return normalizePhoneNumber(contact.phones.first.number);
}

String normalizePhoneNumber(String phone) {
  // Remove all non-digit characters
  return phone.replaceAll(RegExp(r'\D'), '');
}

String? extractPrimaryEmail(FlutterContact contact) {
  if (contact.emails.isEmpty) return null;
  return contact.emails.first.address;
}

DateTime? extractBirthday(FlutterContact contact) {
  if (contact.events.isEmpty) return null;
  final birthdayEvent = contact.events.firstWhere(
    (event) => event.label == EventLabel.birthday,
    orElse: () => null,
  );
  return birthdayEvent?.date;
}

Future<String?> uploadProfilePhoto(Uint8List? photoBytes) async {
  if (photoBytes == null) return null;
  
  // Upload to Supabase Storage
  final fileName = '${uuid.v4()}.jpg';
  final path = 'profile_photos/$fileName';
  
  await supabase.storage
      .from('contact-photos')
      .uploadBinary(path, photoBytes);
  
  return supabase.storage
      .from('contact-photos')
      .getPublicUrl(path);
}

bool hasContactChanged(Contact dbContact, FlutterContact deviceContact) {
  return dbContact.fullName != deviceContact.displayName ||
         dbContact.email != extractPrimaryEmail(deviceContact) ||
         dbContact.birthday != extractBirthday(deviceContact);
  // Photo comparison omitted for simplicity (could hash compare)
}
```

### 3.5 State Management (Riverpod)

```dart
// lib/providers/contact_providers.dart

import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:vip/models/contact.dart';
import 'package:vip/services/contact_sync_service.dart';

// Contact Sync Service Provider
final contactSyncServiceProvider = Provider<ContactSyncService>((ref) {
  return ContactSyncService(
    supabaseClient: ref.watch(supabaseClientProvider),
  );
});

// Contact List State Notifier
final contactListProvider = StateNotifierProvider<ContactListNotifier, AsyncValue<List<Contact>>>((ref) {
  final syncService = ref.watch(contactSyncServiceProvider);
  return ContactListNotifier(syncService);
});

class ContactListNotifier extends StateNotifier<AsyncValue<List<Contact>>> {
  final ContactSyncService _syncService;
  
  ContactListNotifier(this._syncService) : super(const AsyncValue.loading()) {
    loadContacts();
  }
  
  Future<void> loadContacts() async {
    state = const AsyncValue.loading();
    state = await AsyncValue.guard(() async {
      return await _syncService.fetchAllContacts();
    });
  }
  
  Future<void> syncContacts() async {
    await _syncService.performSync();
    await loadContacts();
  }
  
  Future<void> toggleVipStatus(String contactId, bool currentIsVip) async {
    await _syncService.toggleVipStatus(contactId, !currentIsVip);
    await loadContacts();
  }
}

// Search Query Provider
final searchQueryProvider = StateProvider<String>((ref) => '');

// Filtered VIP Contacts Provider
final vipContactsProvider = Provider<List<Contact>>((ref) {
  final contacts = ref.watch(contactListProvider).value ?? [];
  final searchQuery = ref.watch(searchQueryProvider).toLowerCase();
  
  return contacts
      .where((c) => c.isVip && c.status == 'active')
      .where((c) => 
          searchQuery.isEmpty ||
          c.fullName.toLowerCase().contains(searchQuery) ||
          c.phoneNumber.contains(searchQuery))
      .toList()
    ..sort((a, b) => a.fullName.compareTo(b.fullName));
});

// Filtered All Contacts Provider (non-VIP)
final allContactsProvider = Provider<List<Contact>>((ref) {
  final contacts = ref.watch(contactListProvider).value ?? [];
  final searchQuery = ref.watch(searchQueryProvider).toLowerCase();
  
  return contacts
      .where((c) => !c.isVip && c.status == 'active')
      .where((c) => 
          searchQuery.isEmpty ||
          c.fullName.toLowerCase().contains(searchQuery) ||
          c.phoneNumber.contains(searchQuery))
      .toList()
    ..sort((a, b) => a.fullName.compareTo(b.fullName));
});

// Sync Status Provider
final syncStatusProvider = StateProvider<bool>((ref) => false);
```

### 3.6 Security & Privacy

#### Row Level Security (RLS)
- **Enforced:** All contacts table queries filtered by `auth.uid() = user_id`
- **Policies:** Separate policies for SELECT, INSERT, UPDATE, DELETE operations
- **Isolation:** Users can only access their own contacts

#### Data Privacy
- **Contact Photos:** Stored in Supabase Storage with RLS policies
- **Phone Numbers:** Normalized and stored securely, never exposed in logs
- **Platform Contact IDs:** Stored for sync matching, not shared externally

#### Permission Handling
- **iOS Permissions:** Clear explanation before requesting access
- **Permission Denial:** Graceful fallback with settings deep-link
- **No Silent Failures:** Always inform user if permissions blocked

### 3.7 Performance Considerations

#### Optimization Strategies
1. **Virtualized Lists:** Use `ListView.builder` for efficient rendering of large contact lists
2. **Debounced Search:** Debounce search input by 300ms to reduce re-renders
3. **Cached Contact List:** Cache contact list locally (Hive/SharedPreferences) for instant UI load
4. **Lazy Photo Loading:** Load profile photos asynchronously with placeholder
5. **Batch Operations:** Batch contact inserts/updates for initial sync
6. **Background Sync:** Perform sync in background thread to avoid UI blocking

#### Performance Targets
- **Initial Load:** < 1 second for UI render with cached data
- **Search Response:** < 100ms for search results display
- **VIP Toggle:** < 500ms for status update and UI refresh
- **Sync Duration:** < 5 seconds for 100 contacts, < 15 seconds for 500 contacts

### 3.8 Auth Integration Touchpoints
- **Session Source of Truth:** `ContactSyncService` must request Supabase clients via the same Riverpod providers defined in the Auth/Login spec so session refresh, logout, and biometric unlock states stay centralized.
- **Auth State Listener:** Subscribe to `supabase.auth.onAuthStateChange` and immediately:
  1. Cancel in-flight sync jobs when `session == null`
  2. Clear cached contact lists from local storage
  3. Navigate the user back to the auth stack defined in the Auth/Login spec
- **Onboarding Gate:** Only start `performInitialSync()` when `profiles.onboarding_completed_at` is non-null; otherwise the auth experience continues showing the onboarding tutorial (avoids the race condition called out in the Auth spec).
- **Secure Storage Hygiene:** Follow the auth spec’s guidance that logout/account deletion clears secure storage; contact modules must treat resulting empty caches as expected and avoid showing stale contacts.
- **Biometric Re-Entry:** When biometrics are enabled, defer sync until the biometric prompt succeeds to ensure contacts never appear over a locked screen.

---

## 4. Implementation Phases

### Phase 1: Database & Core Infrastructure (Priority: Critical)
**Duration:** 1-2 days

**Tasks:**
- Create `contacts` table migration
- Implement RLS policies
- Set up Supabase Storage bucket for contact photos
- Create `Contact` data model (Freezed)
- Implement basic CRUD operations via Supabase client

**Acceptance Criteria:**
- Database schema deployed to Supabase
- RLS policies tested and enforced
- Contact model generated with Freezed
- Basic insert/update/delete operations working

### Phase 2: Contact Sync Service (Priority: Critical)
**Duration:** 2-3 days

**Tasks:**
- Integrate `flutter_contacts` plugin
- Implement iOS permission request flow
- Build `ContactSyncService` with initial sync logic
- Build incremental sync with change detection
- Implement phone number normalization
- Add profile photo upload to Supabase Storage
- Create contact sync error handling
- Wire `ContactSyncService` into the Auth/Login session providers so sync waits for verified, unlocked sessions before running

**Acceptance Criteria:**
- Permission request displays with explanation
- Initial sync imports all device contacts
- Incremental sync detects and updates changed contacts
- Phone numbers normalized correctly
- Profile photos uploaded and accessible
- Errors handled gracefully with user feedback
- Sync automatically pauses and resumes based on Auth/Login session events

### Phase 3: State Management & Providers (Priority: Critical)
**Duration:** 1-2 days

**Tasks:**
- Create Riverpod providers for contact state
- Implement `ContactListNotifier` with async state
- Create search query provider
- Create filtered VIP/All contacts providers
- Add sync status provider for loading states

**Acceptance Criteria:**
- Contact list loads via Riverpod provider
- VIP/All contacts filters work correctly
- Search filters contacts in real-time
- Loading states managed properly
- State updates trigger UI re-renders

### Phase 4: UI Components (Priority: High)
**Duration:** 3-4 days

**Tasks:**
- Build `VipManagementScreen` main screen
- Create `ContactListSection` reusable widget
- Build `ContactCard` widget with profile photo, name, phone
- Implement `SearchBar` widget with debounced search
- Add alphabetical grouping (section headers)
- Implement pull-to-refresh functionality
- Add empty state messages
- Style components with app theme

**Acceptance Criteria:**
- VIP Management screen displays both sections
- Contact cards render with photos and info
- Search filters contacts across both sections
- Alphabetical grouping displays correctly
- Pull-to-refresh triggers sync
- Empty states show helpful messages
- UI matches design specifications

### Phase 5: VIP Management Actions (Priority: High)
**Duration:** 1-2 days

**Tasks:**
- Implement tap handler to toggle VIP status
- Add visual feedback for VIP state (star icon)
- Implement contact section movement animation
- Add loading indicators during updates
- Implement error handling for failed updates

**Acceptance Criteria:**
- Tapping contact toggles VIP status
- Contact moves between sections smoothly
- Visual indicator shows VIP status clearly
- Loading state shows during update
- Errors display user-friendly messages

### Phase 6: Background Sync & Auto-Update (Priority: Medium)
**Duration:** 1-2 days

**Tasks:**
- Implement background sync on app launch
- Add periodic sync (optional, based on iOS background limitations)
- Create sync scheduling logic
- Add conflict resolution for concurrent edits
- Implement silent update notifications (in-app only)

**Acceptance Criteria:**
- Contacts sync automatically on app launch
- Background sync completes silently
- Conflicts resolved without data loss
- No disruptive notifications during sync

### Phase 7: Edge Cases & Polish (Priority: Medium)
**Duration:** 2-3 days

**Tasks:**
- Handle duplicate name disambiguation
- Handle contacts without phone numbers (email fallback)
- Optimize large contact list performance (virtualization)
- Add contact list caching for offline access
- Implement graceful degradation for sync failures
- Add retry logic with exponential backoff
- Polish animations and transitions

**Acceptance Criteria:**
- Duplicate names show phone numbers clearly
- Email-only contacts handled (if supported)
- Large lists (500+ contacts) render smoothly
- Cached contacts load instantly
- Sync failures retry automatically
- Animations smooth and polished

### Phase 8: Testing & QA (Priority: High)
**Duration:** 2-3 days

**Tasks:**
- Write widget tests for UI components
- Write unit tests for sync logic
- Write integration tests for database operations
- Test permission flows on physical iOS device
- Test with various contact list sizes (10, 100, 500+)
- Test duplicate name scenarios
- Test sync with airplane mode / poor network
- Test photo upload edge cases (large files, missing photos)
- Manual QA across different iOS versions

**Acceptance Criteria:**
- 80%+ code coverage for critical paths
- All edge cases tested and handled
- No crashes during sync operations
- Performance meets targets on real devices
- Permission flows work correctly
- UI renders correctly on various screen sizes

---

## 5. Testing Strategy

### 5.1 Unit Tests
**Coverage Areas:**
- Phone number normalization logic
- Contact change detection algorithm
- Email extraction from device contacts
- Birthday parsing logic
- Contact deduplication logic

**Example Test Cases:**
```dart
test('normalizePhoneNumber removes formatting', () {
  expect(normalizePhoneNumber('(555) 123-4567'), equals('5551234567'));
  expect(normalizePhoneNumber('+1-555-123-4567'), equals('15551234567'));
});

test('hasContactChanged detects name updates', () {
  final dbContact = Contact(..., fullName: 'John Smith');
  final deviceContact = FlutterContact(..., displayName: 'Jonathan Smith');
  expect(hasContactChanged(dbContact, deviceContact), isTrue);
});
```

### 5.2 Widget Tests
**Coverage Areas:**
- `ContactCard` renders correctly with/without photo
- `ContactListSection` displays alphabetical grouping
- `SearchBar` filters contacts on input
- VIP toggle updates UI state
- Empty states display appropriate messages

**Example Test Cases:**
```dart
testWidgets('ContactCard displays name and phone', (tester) async {
  await tester.pumpWidget(
    MaterialApp(
      home: ContactCard(
        contact: Contact(fullName: 'John Smith', phoneNumber: '5551234567'),
      ),
    ),
  );
  
  expect(find.text('John Smith'), findsOneWidget);
  expect(find.text('(555) 123-4567'), findsOneWidget);
});

testWidgets('Search filters contacts in real-time', (tester) async {
  // Setup contacts provider with test data
  // Render VipManagementScreen
  // Type into search bar
  // Verify filtered results appear
});
```

### 5.3 Integration Tests
**Coverage Areas:**
- Full sync flow (fetch device contacts → insert to DB → display in UI)
- VIP toggle flow (tap contact → update DB → move sections)
- Search flow (type query → filter → display results)
- Permission flow (request → grant/deny → handle result)
- Auth session gating (logout or expired session removes contact data and routes user back to Auth/Login experience)

**Example Test Cases:**
```dart
testWidgets('Full sync flow imports contacts', (tester) async {
  // Mock flutter_contacts plugin
  // Mock Supabase client
  // Trigger sync
  // Verify contacts inserted to DB
  // Verify UI displays contacts
});
```

### 5.4 Manual Testing Checklist
- [ ] Initial permission request displays clear message
- [ ] Permission denial shows settings deep-link
- [ ] Initial sync imports all device contacts
- [ ] Contact sync only starts after Auth/Login onboarding completion
- [ ] Contact cards display profile photos correctly
- [ ] Alphabetical grouping works for all sections
- [ ] Search filters across both sections
- [ ] VIP toggle moves contact between sections
- [ ] Pull-to-refresh triggers sync with loading indicator
- [ ] Large contact lists (500+) render without lag
- [ ] Duplicate names show phone numbers for disambiguation
- [ ] Contact updates from device sync silently
- [ ] Sync failures show user-friendly error messages
- [ ] Offline mode shows cached contacts
- [ ] Re-opening app after contact edits syncs changes
- [ ] Logging out or expiring the session via Auth/Login immediately hides contacts and requires re-authentication

---

## 6. Dependencies & Requirements

### 6.1 Flutter Packages
```yaml
dependencies:
  flutter:
    sdk: flutter
  
  # State Management
  flutter_riverpod: ^2.4.0
  riverpod_annotation: ^2.3.0
  
  # Data Models
  freezed_annotation: ^2.4.1
  json_annotation: ^4.8.1
  
  # Backend
  supabase_flutter: ^2.0.0
  
  # Contact Access
  flutter_contacts: ^1.1.7
  
  # Permissions
  permission_handler: ^11.0.1
  
  # Utilities
  uuid: ^4.1.0
  intl: ^0.18.1

dev_dependencies:
  flutter_test:
    sdk: flutter
  
  # Code Generation
  build_runner: ^2.4.6
  freezed: ^2.4.5
  json_serializable: ^6.7.1
  riverpod_generator: ^2.3.0
  
  # Testing
  mockito: ^5.4.2
  flutter_lints: ^3.0.0
```

### 6.2 iOS Permissions (Info.plist)
```xml
<key>NSContactsUsageDescription</key>
<string>VIP needs access to your contacts to help you manage relationships with the people who matter most.</string>
```

### 6.3 Supabase Requirements
- **Database:** PostgreSQL with UUID extension enabled
- **Storage:** Public bucket for contact profile photos
- **Auth:** Supabase Auth configured with user authentication

### 6.4 Platform Requirements
- **iOS:** Minimum version 13.0+ (for flutter_contacts plugin)
- **Android:** Deferred to future phase

---

## 7. Risks & Mitigations

### 7.1 Technical Risks

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|-----------|
| **Contact permission denial** | High - Core feature unusable | Medium | Clear explanation before request, settings deep-link, graceful fallback |
| **Large contact lists cause performance issues** | Medium - Poor UX for power users | Medium | Implement virtualized lists, lazy photo loading, pagination |
| **Phone number format variations cause duplicate detection failures** | High - Data integrity issues | Low | Robust normalization logic, comprehensive testing |
| **iOS background sync limitations** | Medium - Contacts may be stale | High | Manual refresh button, sync on app launch |
| **Contact photo upload failures** | Low - Cosmetic issue | Low | Retry logic, fallback to placeholder icon |
| **Sync conflicts (concurrent edits)** | Medium - Data loss potential | Low | Timestamp-based conflict resolution, preserve DB as source of truth |

### 7.2 Product Risks

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|-----------|
| **Users don't understand VIP concept** | High - Low adoption | Low | Clear onboarding, intuitive UI with visual indicators |
| **Users add too many VIPs** | Medium - Dilutes feature value | Medium | No hard limit, but consider "suggested VIP count" messaging |
| **Users forget to sync manually** | Medium - Stale data | Low | Automatic background sync on launch, notification reminders (future) |

### 7.3 Timeline Risks

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|-----------|
| **iOS contact plugin integration issues** | High - Blocks critical path | Low | Early spike/proof-of-concept, backup plugin options |
| **Supabase RLS policy debugging takes longer than expected** | Medium - Delays security testing | Medium | Allocate extra time, use Supabase CLI testing tools |
| **Performance optimization takes longer than planned** | Medium - Delays polish phase | Medium | Defer non-critical optimizations to future iterations |

---

## 8. Open Questions & Future Considerations

### 8.1 Open Questions
- **Q1:** Should we support email-only contacts (no phone number) in initial release?
  - **Decision Needed By:** End of Phase 2
  - **Impact:** Medium - affects sync logic complexity
  
- **Q2:** What's the maximum number of VIP contacts we should recommend to users?
  - **Decision Needed By:** End of Phase 4 (UI phase)
  - **Impact:** Low - informational only, no technical constraint

- **Q3:** Should we show a count of unsynced changes or last sync timestamp?
  - **Decision Needed By:** End of Phase 6
  - **Impact:** Low - nice-to-have feature

### 8.2 Future Enhancements (Out of Current Scope)

#### Android Support
- Implement Android contact sync using same database schema
- Add `platform_source = 'android'` support
- Handle Android-specific permissions

#### Contact Groups
- Sync contact groups from device
- Allow filtering VIPs by group
- Display group membership in UI

#### Contact Editing
- Allow users to edit contact info within app
- Sync edits back to device (iOS limitations may apply)
- Track edit history

#### Advanced Search
- Search by birthday month
- Search by relationship closeness level (future integration)
- Search by last interaction date

#### Merge Contacts
- Detect potential duplicates (same name, different phones)
- UI to merge duplicate contacts
- Preserve all historical data during merge

#### Bulk Actions
- Select multiple contacts to mark as VIP
- Import VIPs from CSV
- Export VIP list

#### Contact Insights
- Show VIP contact statistics (count, avg interactions, etc.)
- Show contacts with missing information (no birthday, no email)
- Suggest VIPs based on interaction frequency

---

## 9. Success Criteria & Acceptance

### 9.1 Feature Completion Checklist
- [ ] Database schema deployed with RLS policies
- [ ] Contact sync service implemented (initial + incremental)
- [ ] iOS permission flow working
- [ ] Contact sync gated behind verified Auth/Login sessions and onboarding completion
- [ ] VIP Management screen displays both sections
- [ ] Search filters contacts in real-time
- [ ] VIP toggle moves contacts between sections
- [ ] Pull-to-refresh triggers manual sync
- [ ] Background sync on app launch
- [ ] Profile photos display correctly
- [ ] Alphabetical grouping implemented
- [ ] Duplicate name disambiguation working
- [ ] Large contact lists render smoothly (500+ contacts)
- [ ] All edge cases handled (permission denial, sync failures)
- [ ] Unit tests written (80%+ coverage)
- [ ] Widget tests written for all UI components
- [ ] Integration tests written for critical flows
- [ ] Manual QA completed on physical iOS device
- [ ] Performance targets met (load times, sync duration)

### 9.2 Launch Criteria
**Must Have (P0 - Blocking Launch):**
- ✅ Contact sync works reliably for 90%+ of users
- ✅ VIP management UI is intuitive and bug-free
- ✅ RLS policies prevent unauthorized access
- ✅ No data loss during sync operations
- ✅ Permission flows follow iOS best practices
- ✅ Performance meets targets on iPhone 11+ devices

**Should Have (P1 - Launch but Track Issues):**
- ⚠️ Contact photos load reliably
- ⚠️ Duplicate name handling works for common cases
- ⚠️ Sync errors display helpful messages
- ⚠️ Large contact lists (500+) render without major lag

**Nice to Have (P2 - Post-Launch Iteration):**
- 💡 Sync timestamp displayed in UI
- 💡 Offline mode with cached contacts
- 💡 Contact insights/statistics

### 9.3 Post-Launch Monitoring
**Metrics to Track:**
- Contact sync success rate (target: 99%+)
- Average sync duration (target: <5s for 100 contacts)
- VIP toggle success rate (target: 99.9%+)
- Permission grant rate (target: 80%+)
- Average VIPs per user (target: 10-50)
- User retention after adding first VIP (target: 70%+)

**Error Monitoring:**
- Track sync failures by error type (permission, network, database)
- Monitor photo upload failures
- Alert on RLS policy violations (should be zero)

---

## 10. Appendix

### 10.1 Technical References
- **Flutter Contacts Plugin:** https://pub.dev/packages/flutter_contacts
- **Supabase RLS Guide:** https://supabase.com/docs/guides/auth/row-level-security
- **Riverpod State Management:** https://riverpod.dev/docs/introduction/getting_started
- **iOS Contact Permissions:** https://developer.apple.com/documentation/contacts

### 10.2 Related Documentation
- **Product Mission:** `/agent-os/product/mission.md`
- **Tech Stack Standards:** `/agent-os/standards/global/tech-stack.md`
- **Database Standards:** `/agent-os/standards/backend/models.md`
- **Component Standards:** `/agent-os/standards/frontend/components.md`
- **Error Handling Standards:** `/agent-os/standards/global/error-handling.md`

### 10.3 Revision History
| Date | Version | Changes | Author |
|------|---------|---------|--------|
| 2025-11-16 | 1.0 | Initial specification created | Spec Writer Agent |

---

**End of Specification Document**

