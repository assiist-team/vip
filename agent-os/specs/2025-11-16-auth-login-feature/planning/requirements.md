# Spec Requirements: Auth/Login Feature

## Initial Description
auth/login feature

## Requirements Discussion

### First Round Questions

**Q1:** I assume we’ll leverage Supabase Auth with email-based passwordless sign-in (magic links or OTP) as the first authentication method so we can ship quickly. Is that correct, or do you want a traditional email + password flow from day one?
**Answer:** Mirror the prior user-auth spec: use Supabase email/password plus Google OAuth, require email verification, and keep passwordless flows out of scope for now.

**Q2:** I’m thinking the app should always show an authentication gate before any data sync, then persist the Supabase session token securely (e.g., flutter_secure_storage) to auto-rehydrate on relaunch. Should we force a fresh login after a set inactivity window instead?
**Answer:** Follow the previous approach: gate everything behind auth, persist Supabase sessions securely for automatic rehydration, and only require re-login when a session expires or the user logs out.

**Q3:** Should we include social/OAuth providers (Apple, Google) in the initial release, or keep those as stubs/placeholders until after the MVP validates the passwordless experience?
**Answer:** Match the earlier spec exactly—ship with Google Sign-In only (Apple deferred) alongside email/password.

**Q4:** For error handling, I’m assuming we’ll map Supabase Auth errors to friendly, actionable messages (e.g., retry hints, connectivity status) and log the raw errors to Sentry. Is that the right level of transparency, or do you prefer a more minimal UX with generic “Try again” prompts?
**Answer:** Mirror that pattern: user-facing errors stay concise and actionable while raw Supabase/Auth exceptions go to Sentry for diagnostics.

**Q5:** To ensure downstream features (contact sync, notes) always have a valid user context, should the login flow verify that required onboarding metadata (e.g., VIP list seed data) exists before granting access, or can that happen lazily after login?
**Answer:** Same as before—once authenticated the user can enter the app, then an onboarding/tutorial sequence runs immediately to gather any remaining info; no extra gating before login completes.

**Q6:** Are there any explicit exclusions for this first iteration (e.g., MFA, account deletion, session revocation screens) that we should document to avoid scope creep?
**Answer:** Document the same exclusions list: no MFA, phone auth, account linking, passwordless, or social profile imports; include account deletion, change password, name update, logout, and biometric toggle just like the referenced spec.

### Existing Code to Reference

**Similar Features Identified:**
- Feature: User Authentication & Profile Spec (reference implementation decisions) - Path: `/Users/benjaminmackenzie/Dev/memories/agent-os/specs/2025-11-16-user-auth-and-profile`
- Components to potentially reuse: All auth flows, onboarding tutorial structure, and settings surface defined in that spec (UI/widget patterns mirrored).
- Backend logic to reference: Supabase Auth integration details and account deletion flow documented in the referenced spec.

### Follow-up Questions
None.

## Visual Assets

No visual assets provided.

## Requirements Summary

### Functional Requirements
- Supabase email/password authentication with required email verification.
- Google OAuth Sign-In alongside email/password; Apple and other providers deferred.
- Supabase password reset flow via email-based reset links.
- Optional biometric authentication (Face ID/Touch ID/fingerprint) prompted after first successful login and controllable via settings.
- User profile captures `Name` (required) plus read-only email from auth provider; no profile photos or extra fields yet.
- Basic settings include change password, update name, toggle biometric login, log out, and account deletion with multi-step confirmation.
- First-time onboarding/tutorial shown right after initial login/signup, not as a pre-auth gate.
- Entire app remains behind the auth gate; Supabase sessions persist securely via `flutter_secure_storage` and rehydrate automatically.

### Reusability Opportunities
- Mirror UX, flows, and Supabase integration patterns documented in `/Users/benjaminmackenzie/Dev/memories/agent-os/specs/2025-11-16-user-auth-and-profile`.
- Reuse any biometrics, onboarding, and settings components derived from that prior work.
- Follow the same backend/account deletion flows already defined for the earlier auth/profile scope.

### Scope Boundaries
**In Scope:**
- Auth flows (email/password + Google) with email verification and password reset.
- Biometric opt-in, onboarding tutorial, and essentials-only profile/settings management.
- Account deletion capability meeting App Store/GDPR expectations.

**Out of Scope:**
- Passwordless magic links or OTP flows.
- Additional OAuth providers (Apple, etc.).
- MFA, phone-number auth, account linking/merging, social profile imports.
- Profile photos or extra profile fields, notification preferences, or theme settings.

### Technical Considerations
- Use Supabase Auth + PostgREST plus Flutter (`riverpod`, `flutter_secure_storage`, `local_auth`).
- Relay Supabase errors to Sentry while displaying friendly UI copy.
- Ensure RLS policies restrict data to the authenticated user.
- Follow onboarding/tutorial UX guidelines from the referenced spec to keep the app consistent.
