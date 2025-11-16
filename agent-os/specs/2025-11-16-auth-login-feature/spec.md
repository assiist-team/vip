# Specification: Auth/Login Feature

## Goal
Deliver a secure, Supabase-backed authentication gate that supports email/password plus Google sign-in, persists verified sessions safely, and hands users into onboarding with essential profile and settings management available post-login.

## User Stories
- As a new user, I want to create an account with email/password or Google so that I can start using the app immediately after verifying my email.
- As a returning user, I want the app to remember my session and optionally unlock with biometrics so that I can re-enter quickly without retyping credentials unless security requires it.
- As a privacy-minded user, I want clear settings for changing my password, toggling biometrics, logging out, or deleting my account so that I stay in control of my data.

## Dependencies & Downstream Contracts
- **Contact Sync & VIP List Management (`2025-11-16-contact-sync-vip-list-management`):** Contact fetching never begins until this auth flow produces a verified `supabase.auth.currentSession`, biometric unlock succeeds (when enabled), and `profiles.onboarding_completed_at` is set. Auth logout or session expiry must broadcast through the shared Riverpod Supabase client so contact modules pause sync, clear cached contacts, and route users back to the auth stack per that spec.
- **Future Core Features (e.g., Notes, Relationship Summaries):** Reuse the same session providers, secure storage entries, and auth-state listeners so each downstream feature inherits consistent gating without duplicating credential logic.
- **Edge Function Ownership:** Account deletion tears down `auth.users`, `profiles`, and dependent tables, so downstream specs (contacts, notes) rely on this feature to cascade deletes safely; auth code must emit completion events so those teams can treat missing rows as expected.

## Specific Requirements

**Authentication Entry Points**
- Use a shared Supabase client (anon key) with Riverpod providers to handle signup, login, logout, and session refresh flows.
- Support email/password registration and login plus Google OAuth via `signInWithOAuth(provider: Provider.google)`; Apple and other providers remain deferred.
- Block access to any app surface until `supabase.auth.currentSession` is valid; route unauthenticated users to a dedicated auth stack.
- Require email verification before unlocking the main shell; keep unverified accounts in a waiting screen with CTA to resend verification.
- Surface inline validation for invalid credentials, throttling, or network loss, matching tone from mission/brand guidance.

**Email Verification & Password Recovery**
- Enforce email verification by polling Supabase or prompting users to reopen the magic link; automatically transition once `user.emailConfirmedAt` is present.
- Provide “Resend verification email” controls that respect Supabase rate limits and show success/error states.
- Offer a “Forgot password?” flow that triggers Supabase password reset emails and handles deep-link callbacks to re-open the login screen.
- Apply password validation (min 8 chars, mix of cases or numbers) per `standards/global/validation.md` before submitting to Supabase.
- Log Supabase auth errors (e.g., `InvalidCredentials`, `RateLimitExceeded`) to Sentry with user ID context while keeping user copy concise.

**Session Persistence & Rehydration**
- Persist Supabase session tokens via `flutter_secure_storage`; never store secrets in plain preferences.
- Rehydrate sessions on app launch by checking secure storage first, then `supabase.auth.refreshSession()` to avoid flashing the auth gate.
- Listen to `supabase.auth.onAuthStateChange` to detect expirations or revocations and redirect to login when `session == null`.
- Only prompt for credentials again when a session refresh fails or the user has explicitly logged out; no arbitrary inactivity timeouts.
- Clear secure storage entries whenever logout or account deletion completes.

**Biometric Authentication Opt-In**
- Detect biometric availability through Flutter `local_auth`; prompt after the first successful login/signup (post-verification).
- Store `biometric_enabled` both in the `profiles` table and `flutter_secure_storage` to keep server truth plus local enforcement.
- When enabled, show biometric prompt before hitting Supabase; on failure, fall back to password entry without locking the account.
- Provide settings toggle to enable/disable biometrics; disabling wipes biometric tokens from secure storage.
- Respect accessibility guidance (fallback PIN option, clear messaging) when biometrics are unavailable or denied.

**Profile & Settings Surface**
- Maintain a `profiles` table keyed to `auth.users.id` storing `name text not null`, `biometric_enabled boolean default false`, timestamps, and onboarding markers.
- Require a valid `Name` during signup or immediately after verification; show email as read-only everywhere.
- Settings screen must expose: Edit Name, Change Password (Supabase `updateUser`), Biometric toggle, Log Out, Delete Account.
- Reflect the last sign-in timestamp from Supabase session metadata for transparency.
- Follow global accessibility and responsive standards for touch targets ≥44px and text contrast.

**Onboarding & Tutorial Hand-off**
- After first verified login, launch a three-screen onboarding/tutorial describing value pillars; block the main shell until completion.
- Persist `onboarding_completed_at` in `profiles`; only rerun onboarding when null.
- Provide CTA options such as “Start managing VIPs” versus “Explore later” that route into the app shell or contacts context.
- Trigger any downstream data sync (contacts, notes) only after onboarding completes to avoid race conditions with onboarding prompts.

**Account Deletion & Logout**
- Implement logout that clears Supabase session, secure storage tokens, and Riverpod providers before returning to the intro/auth stack.
- Account deletion requires multi-step confirmations: warning screen → credential re-check (password or Google reauth) → final confirm.
- Execute deletion via a Supabase Edge Function using the service-role key to remove `auth.users`, cascade `profiles`, and scrub user-owned rows.
- Send a confirmation email and display final success toast before tearing down local cache; ensure analytics log the deletion event.

**Error Handling & Observability**
- Map Supabase/Auth error codes to user-friendly copy with remediation (retry, check email, contact support).
- Capture raw errors plus device info to Sentry; include breadcrumbs for auth steps taken.
- Show offline banners or inline states when network connectivity drops before hitting Supabase APIs.
- Provide telemetry events for login success/failure, biometric opt-in/out, password resets, and account deletion to inform future iteration.

**Security & Compliance**
- Enforce Postgres RLS so every `profiles` and downstream table query scopes to `auth.uid()`; never rely solely on client filtering.
- Never log passwords, tokens, or biometric secrets; redact sensitive fields per `standards/global/security.md`.
- Use HTTPS deep links / Universal Links for verification and reset callbacks; validate tokens before mutating state.
- Ensure accessibility compliance (voice-over labels, error text tied to inputs) and keep copy aligned with mission tone.
- Document recovery paths for locked/disabled accounts and ensure support can manually verify via Sentry logs.

## Session Lifecycle Diagram
```
┌──────────────┐   Credentials+OAuth    ┌──────────────────────┐
│ App Launch   │──────────────────────▶│ Auth Stack (Login UI) │
└──────┬───────┘                        └─────────┬────────────┘
       │ Secure storage rehydrate                   │ Email verified?
       │                                            ▼
       │                           ┌──────────────────────────────┐
       │                           │ Verified Session Established │
       │                           └──────────┬───────────────────┘
       │ Biometric enabled?                   │
       ▼                                      ▼
┌────────────────┐                 ┌──────────────────────────────┐
│ Biometric Gate │───────────────▶ │ Onboarding / Tutorial Stack  │
└──────┬─────────┘ completed?      └──────────┬───────────────────┘
       │                                      │ `profiles.onboarding_completed_at`
       ▼                                      ▼
┌──────────────────────────────┐   Broadcast session + onboarding ready
│ Authenticated App Shell      │─────────────────────────────────────────┐
└─────────────┬────────────────┘                                         │
              │ Auth state listener                                      │
              ▼                                                          │
     ┌──────────────────────────┐                                        │
     │ Contact Sync & VIP Mgmt  │◀───────────────────────────────────────┘
     └──────────────────────────┘
```

**Key Callouts:**
- Secure storage → Supabase refresh happens before the UI decides whether to show the auth stack or shell, preventing flicker.
- Biometric prompts fire only after a valid Supabase session exists; failure returns the user to the login UI without leaking session tokens.
- Onboarding completion is the explicit signal for downstream specs (contacts, notes) to start background syncs; they must subscribe to the auth provider so logout/session expiry tears everything back down.
- Logout, session revocation, or account deletion inject a `session == null` event that forces the flow to jump back to the Auth Stack, ensuring no feature can render user data without re-authentication.

## Visual Design
No visual assets provided; follow mission and standards docs for typography, color, and layout decisions.

## Existing Code to Leverage

**`/Users/benjaminmackenzie/Dev/memories/agent-os/specs/2025-11-16-user-auth-and-profile/spec.md`**
- Mirrors the exact auth scope (email/password + Google, biometrics, settings) and provides detailed flow decisions to replicate.
- Describes the `profiles` data model, onboarding gating, and account deletion Edge Function patterns that this spec should inherit.

**`agent-os/specs/2025-11-16-contact-sync-vip-list-management/spec.md`**
- Defines the RLS policies and Supabase row-ownership patterns that downstream contact data already expects post-auth.
- Provides guidance on gating all CRUD APIs by `auth.uid()`, ensuring this auth feature unlocks data consistently.

**`agent-os/standards/global/security.md`**
- Establishes secure session handling, JWT refresh expectations, and logging redaction rules to follow during implementation.

**`agent-os/standards/testing/test-writing.md`**
- Details test coverage expectations for auth-critical flows (happy path, invalid credentials, regression cases) to plan for later tasks.

## Out of Scope
- Passwordless magic links, OTP codes, or any other email-only flows.
- Additional OAuth providers beyond Google (e.g., Apple, Facebook, LinkedIn).
- Multi-factor authentication, phone/SMS-based auth, or device management dashboards.
- Social profile imports, profile photos, bios, or additional profile metadata.
- Notification or theme settings, preference centers, or subscription management.
- Account linking/merging, delegated access, or enterprise SSO.
- Server-driven onboarding personalization, experiments, or dynamic tutorials.
- Session revocation lists, admin tooling, or audit dashboards beyond Sentry logging.

