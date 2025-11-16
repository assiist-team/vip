# Task Breakdown: Contact Sync & VIP List Management

## Overview
- Total Tasks: 47
- Inputs: `spec.md`, `planning/requirements.md`
- Standards to honor: backend (migrations/models/api/queries), frontend (components/css/accessibility/responsive), global (coding-style/commenting/conventions/error-handling/performance/security/validation/tech-stack), testing (`standards/testing/test-writing.md`)
- Session prerequisites: respect `2025-11-16-auth-login-feature` so contact work only runs after a verified `supabase.auth.currentSession` and completed onboarding.

## Task List

### Data Platform & Storage
**Dependencies:** None

- [ ] 1.0 Complete Supabase contact data foundations
  - [ ] 1.1 Write 3-4 focused SQL/unit tests (constraint enforcement, RLS visibility) per `standards/testing/test-writing.md`
  - [ ] 1.2 Author migration for `contacts` table (fields, indexes, triggers) following `standards/backend/migrations.md` and `models.md`
  - [ ] 1.3 Configure `contact-photos` storage bucket + policies aligned with `standards/global/security.md`
  - [ ] 1.4 Implement Freezed `Contact` model + enums + JSON serialization, ensuring naming matches `standards/global/coding-style.md`
  - [ ] 1.5 Validate migrations, RLS, and CRUD by running only the tests from 1.1

**Acceptance Criteria:**
- Migration applies cleanly via Supabase CLI and matches spec schema
- RLS prevents cross-user access while allowing owner CRUD
- Storage bucket enforces private-by-default access with signed URL helper
- Freezed model builds without analyzer warnings

### Device Permissions & Sync Service
**Dependencies:** Task Group 1

- [ ] 2.0 Build Flutter contact permission + sync service layer
  - [ ] 2.1 Write 3-5 focused service tests (phone normalization, dedupe, change detection) using mocks only
  - [ ] 2.2 Integrate `flutter_contacts` + `permission_handler` requesting access per iOS guidelines with copy from spec
  - [ ] 2.3 Implement initial sync pipeline (batch inserts, phone-number-as-truth, platform metadata) using Supabase client
  - [ ] 2.4 Implement incremental sync (diff detection, inactive marking, conflict resolution timestamps)
  - [ ] 2.5 Add structured error handling, retry hooks, and manual refresh trigger integration
  - [ ] 2.6 Wire `ContactSyncService` into the Auth/Login session providers so syncs only start once the shared Supabase client exposes a valid session and biometric gate has succeeded
  - [ ] 2.7 Subscribe to `supabase.auth.onAuthStateChange` to pause in-flight syncs, clear cached contacts, and route users back through the auth stack when `session == null`, and block `performInitialSync` until `profiles.onboarding_completed_at` is set
  - [ ] 2.8 Run ONLY the tests from 2.1 to verify service logic

**Acceptance Criteria:**
- Permission prompts show clear rationale + settings deep link when denied
- Initial sync imports contacts once without duplicates
- Incremental sync updates name/email/photo/birthday silently and marks removed non-VIPs inactive
- Sync waits for verified sessions plus onboarding completion, pauses/resumes with auth events, clears caches, and routes users through auth screens on logout/biometric failure
- Service surfaces failures via typed errors logged per `standards/global/error-handling.md`

### State Management & Local Data
**Dependencies:** Task Groups 1-2

- [ ] 3.0 Implement Riverpod-based state layer + caching
  - [ ] 3.1 Write 2-4 provider tests (search filtering, VIP/all segmentation, sync status transitions)
  - [ ] 3.2 Implement `ContactListNotifier` with async loading, manual refresh, and optimistic VIP toggles
  - [ ] 3.3 Create derived providers (`vipContactsProvider`, `allContactsProvider`, `searchQueryProvider`, `syncStatusProvider`) with alphabetical grouping helpers
  - [ ] 3.4 Add lightweight cache (Hive/SharedPreferences) respecting `standards/global/performance.md` and data privacy constraints
  - [ ] 3.5 Hook providers and cache hydration to the Auth/Login session notifiers so contact state hydrates only when a session is verified and is cleared immediately on logout/account deletion/biometric lock events
  - [ ] 3.6 Run ONLY the tests from 3.1 to validate provider logic

**Acceptance Criteria:**
- Providers deliver de-duplicated, alphabetized data within 100ms target
- Search filtering is case-insensitive and spans both VIP/non-VIP lists
- Cache hydrates UI instantly when offline, then revalidates with Supabase
- Contact lists never appear during locked/unauthenticated states and are purged immediately once Auth/Login signals logout/account deletion
- State transitions emit loading flags consumed by UI

### VIP Management UI & Interactions
**Dependencies:** Task Groups 2-3

- [ ] 4.0 Build VIP management UI surfaces
  - [ ] 4.1 Write 4-6 widget tests (ContactCard render, VIP toggle, search filtering, empty states)
  - [ ] 4.2 Implement `VipManagementScreen` scaffold with search bar + dual sections
  - [ ] 4.3 Build `ContactListSection` with sticky alphabetical headers + virtualization
  - [ ] 4.4 Build `ContactCard` with photo fallback, VIP indicator, phone/email disambiguation, and semantics labels per `standards/frontend/accessibility.md`
  - [ ] 4.5 Implement debounced `SearchBar` tied to providers, ensuring keyboard and VoiceOver support
  - [ ] 4.6 Implement VIP toggle interactions (tap gesture, haptics, optimistic movement between sections)
  - [ ] 4.7 Add pull-to-refresh + loading indicator + sync error banner messaging
  - [ ] 4.8 Apply theming/responsiveness per `standards/frontend/components.md`, `css.md`, and `responsive.md`
  - [ ] 4.9 Run ONLY the tests from 4.1 to verify UI behaviors

**Acceptance Criteria:**
- Screen matches layout described in spec, including counts and headers
- Tapping contact toggles VIP within 500ms and animates section change
- Search updates results within 300ms debounce window
- Empty, permission-denied, and sync-error states show prescribed copy
- UI passes basic accessibility scan (labels, contrast, hit targets)

### Background Sync, Performance & Telemetry
**Dependencies:** Task Groups 2-4

- [ ] 5.0 Harden sync scheduling, resilience, and metrics
  - [ ] 5.1 Write 2-3 integration tests (background sync trigger, retry logic, manual refresh handshake)
  - [ ] 5.2 Implement app-lifecycle/background sync orchestration with guards against concurrent runs
  - [ ] 5.3 Wire pull-to-refresh + status indicators into state layer with spinner timeouts >2s
  - [ ] 5.4 Add retry/backoff, offline fallback messaging, and silent conflict resolution logging
  - [ ] 5.5 Emit performance + reliability metrics (sync duration, failure types) per `standards/global/performance.md`
  - [ ] 5.6 Run ONLY the tests from 5.1 to confirm resilience behaviors

**Acceptance Criteria:**
- Sync runs automatically on app launch/resume without blocking UI
- Manual refresh surfaces spinner + banner only when needed
- Retries respect exponential backoff and stop after safe limit
- Metrics/logs capture adoption KPIs defined in spec

### Feature QA, Coverage & Release Prep
**Dependencies:** Task Groups 1-5

- [ ] 6.0 Complete cross-discipline QA + release readiness
  - [ ] 6.1 Review tests from groups 1-5 to catalog coverage (approx. 14-28 tests)
  - [ ] 6.2 Identify critical workflow gaps (permission denial, duplicate names, large lists, offline, auth session expiration/biometric lock)
  - [ ] 6.3 Add up to 10 targeted integration/E2E tests focused on high-risk workflows only
  - [ ] 6.4 Run feature-specific suite (tests from 1.1, 2.1, 3.1, 4.1, 5.1, 6.3) and document results
  - [ ] 6.5 Execute manual QA checklist (physical iOS device, 10/100/500+ contacts, photo upload) and log findings in spec folder
  - [ ] 6.6 Record open questions/future work items back into `spec.md` Appendix if unresolved
  - [ ] 6.7 Verify logging out, session expiration, or failing biometric unlock immediately hides contacts and reroutes through the Auth/Login experience

**Acceptance Criteria:**
- Total automated tests for this feature ≤34 and all green
- QA log confirms permission, sync, VIP toggle, search, and error flows across device sizes
- Auth/session edge cases (logout, expiration, biometric failure) verified with no stale data shown
- Any residual risks documented with mitigation/owner
- Release notes highlight background sync limitations + next steps

