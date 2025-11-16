# Task Breakdown: Auth/Login Feature

## Overview
- Total Tasks: 34
- Inputs: `spec.md`, `planning/requirements.md`
- Standards to honor: backend (api/migrations/models/queries), frontend (components/css/accessibility/responsive), global (coding-style/commenting/conventions/error-handling/performance/security/validation/tech-stack), testing (`standards/testing/test-writing.md`)

## Task List

### Supabase Foundations & Data Model
**Dependencies:** None

- [ ] 1.0 Establish Supabase auth + profile scaffolding
  - [ ] 1.1 Configure shared Supabase client (anon key) + Riverpod providers per tech stack doc; capture env/secrets loading approach.
  - [ ] 1.2 Author migration for `profiles` table (`id uuid primary key references auth.users`, `name text not null`, `biometric_enabled boolean default false`, `onboarding_completed_at timestamptz`, timestamps) following `standards/backend/migrations.md`.
  - [ ] 1.3 Add trigger or edge function hook to auto-create `profiles` row on signup; document failure fallbacks.
  - [ ] 1.4 Write RLS policies limiting `profiles` access to `auth.uid()` plus any service-role exceptions; cover CRUD paths mentioned in spec.
  - [ ] 1.5 Create 3-4 SQL/unit tests ensuring migration + RLS behavior per `standards/testing/test-writing.md`.
  - [ ] 1.6 Document profile-related constants/models for Flutter domain layer (Freezed or equivalent).

**Acceptance Criteria**
- Migration applies cleanly; RLS forbids cross-user reads/writes.
- Provider exposes authenticated Supabase client without analyzer warnings.
- Tests cover happy path + unauthorized attempts.

### Authentication Surfaces & Flows
**Dependencies:** Task Group 1

- [ ] 2.0 Build authentication stack (signup, login, OAuth entry)
  - [ ] 2.1 Design screen flow (intro → login/signup tabs) with routing guards blocking app shell until `supabase.auth.currentSession` exists.
  - [ ] 2.2 Implement email/password signup/login forms with validation (min 8 chars, mixed case/number) referencing `standards/global/validation.md`.
  - [ ] 2.3 Integrate Google OAuth via `signInWithOAuth(provider: Provider.google)` including deep link/callback handling on iOS/Android.
  - [ ] 2.4 Add inline error surfaces for invalid credentials, throttling, network loss; ensure copy mirrors mission tone and references `standards/global/error-handling.md`.
  - [ ] 2.5 Create 4-6 widget/service tests covering signup success, wrong password, throttled login, Google cancelation.
  - [ ] 2.6 Wire mission-consistent theming, accessibility labels, and responsive layouts for phone/tablet sizes.

**Acceptance Criteria**
- Unauthenticated users never reach app shell.
- Validation + error states match spec wording and accessibility rules.
- Automated tests pass reliably.

### Email Verification & Password Recovery
**Dependencies:** Task Group 2

- [ ] 3.0 Implement verification holding pattern + password reset
  - [ ] 3.1 Add post-signup waiting screen that polls/reloads session until `emailConfirmedAt` is populated or timeout occurs.
  - [ ] 3.2 Implement “Resend verification email” action respecting Supabase API rate limits; show success/failure toasts.
  - [ ] 3.3 Build deep-link handler for verification + password reset callbacks (iOS/Android) ensuring HTTPS/Universal links usage.
  - [ ] 3.4 Add “Forgot password?” entry from login, calling Supabase reset APIs and presenting confirmation states.
  - [ ] 3.5 Capture verification/forgot-password analytics + Sentry breadcrumbs for debugging.
  - [ ] 3.6 Write 2-3 integration tests simulating verification completion + reset errors.

**Acceptance Criteria**
- Users cannot proceed to onboarding without verified email.
- Resend/reset flows provide actionable copy + logging.
- Deep-link handling works on device and simulator.

### Session Persistence & Biometrics
**Dependencies:** Task Group 2

- [ ] 4.0 Persist/rehydrate sessions plus biometric opt-in
  - [ ] 4.1 Implement secure storage of Supabase sessions via `flutter_secure_storage`; ensure encryption + redaction rules.
  - [ ] 4.2 Add app-launch rehydration logic (secure storage → `refreshSession()`) that avoids flashing auth UI when tokens are valid.
  - [ ] 4.3 Subscribe to `supabase.auth.onAuthStateChange` to handle expiry, revocation, logout.
  - [ ] 4.4 Detect biometric availability via `local_auth`, prompt after first successful login, and persist `biometric_enabled` in both secure storage + `profiles`.
  - [ ] 4.5 Build biometric unlock gate preceding Supabase call, with fallbacks to password if biometric fails or user cancels.
  - [ ] 4.6 Provide settings toggle to remove biometrics, wiping stored tokens and updating profile row.
  - [ ] 4.7 Add 3-4 unit/widget tests covering rehydration success/failure, biometric opt-in/out, and fallback paths.

**Acceptance Criteria**
- Sessions survive relaunch until Supabase expires them.
- Biometric prompt only appears on supported hardware and respects accessibility guidance.
- Logout clears all secure artefacts.

### Onboarding & Post-Login Hand-off
**Dependencies:** Task Groups 1-4

- [ ] 5.0 Build onboarding tutorial triggered after first verified login
  - [ ] 5.1 Implement onboarding navigator (≤3 screens) communicating value pillars, referencing mission doc for tone.
  - [ ] 5.2 Persist `onboarding_completed_at` in `profiles`; guard so onboarding shows only when null.
  - [ ] 5.3 Provide CTA wiring (“Start managing VIPs” / “Explore later”) routing into primary shell and triggering downstream data sync.
  - [ ] 5.4 Defer contact/notes sync until onboarding emits completion to avoid race conditions.
  - [ ] 5.5 Add analytics + Sentry breadcrumbs for onboarding start/completion/cancel.
  - [ ] 5.6 Write 2-3 UI tests verifying gating behavior and CTA routing.

**Acceptance Criteria**
- Users always see onboarding exactly once per account unless reset.
- Downstream services don’t run before onboarding completes.
- Tests prove gating works after reinstall/rehydrate.

### Settings, Logout & Account Deletion
**Dependencies:** Task Groups 1-4

- [ ] 6.0 Deliver settings area with account controls
  - [ ] 6.1 Implement Settings screen (Account, Security, Support sections) per spec using shared UI components.
  - [ ] 6.2 Build “Edit Name” form with validation + Supabase update; email remains read-only.
  - [ ] 6.3 Add “Change Password” path using Supabase `updateUser`, with confirmation states and error surfacing.
  - [ ] 6.4 Implement logout button that clears Supabase session, Riverpod state, secure storage, and navigates to intro.
  - [ ] 6.5 Design multi-step account deletion confirmation (warning → credential check → final confirm) with blocking modals.
  - [ ] 6.6 Build Supabase Edge Function (service-role key) to delete `auth.users`, cascade `profiles`, and scrub dependent tables; include audit logging.
  - [ ] 6.7 Emit confirmation email + in-app toast upon deletion; ensure client cleans caches afterward.
  - [ ] 6.8 Write 4-5 integration tests for logout, name update, password change failure, and deletion success/abort flows.

**Acceptance Criteria**
- Settings actions respect accessibility (labels, focus management, hit areas ≥44px).
- Account deletion only proceeds after credential re-check and server confirmation.
- Edge Function guarded by service role and logs auditing info.

### Observability, Error Handling & Release QA
**Dependencies:** Task Groups 1-6

- [ ] 7.0 Harden telemetry and perform final QA
  - [ ] 7.1 Map Supabase/Auth error codes to user-friendly copy plus remediation tips; centralize in error mapper.
  - [ ] 7.2 Ensure Sentry captures raw exceptions with user/context metadata and redacts secrets.
  - [ ] 7.3 Emit telemetry events for login success/failure, biometric toggles, password reset, onboarding completion, account deletion.
  - [ ] 7.4 Compile E2E/feature test suite (tests from groups 1.5, 2.5, 3.6, 4.7, 5.6, 6.8) and document run instructions.
  - [ ] 7.5 Conduct manual QA checklist (fresh signup, Google login, verification loop, biometric enable/disable, deletion) across at least one iOS and one Android device.
  - [ ] 7.6 Capture open risks/future enhancements back into `spec.md` Appendix or issue tracker.

**Acceptance Criteria**
- Error surfaces stay concise and match accessibility rules.
- Monitoring dashboards/Sentry show key auth funnel metrics.
- QA log indicates pass/fail for all critical flows with mitigation notes.

