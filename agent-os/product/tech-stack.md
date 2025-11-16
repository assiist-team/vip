# VIP App Tech Stack

## Framework & Runtime
- **Application Framework:** Flutter (iOS-first, Android planned for future)
- **Language/Runtime:** Dart (Flutter), Python (backend scripts/tooling)
- **Package Manager:** pub (Flutter/Dart), pip (Python)

## Frontend
- **UI Framework:** Flutter with Material/Cupertino widgets
- **Styling:** Flutter custom themes and widget composition
- **State Management:** Riverpod
- **Voice Input:** In-house Flutter plugin for iOS dictation (native APIs via plugin interface)

## Database & Storage
- **Database:** Supabase (PostgreSQL)
- **Database Operations:** Cursor MCP for Supabase
- **Vector Database:** pgvector extension on Supabase for message rules and semantic search
- **Storage:** Supabase Storage (for any media/attachments)

## Backend & Serverless
- **Cloud Functions:** Supabase Edge Functions
  - Signal extraction from notes
  - Message generation orchestration
  - Scheduled jobs for birthday/event detection
- **API:** Supabase PostgREST (auto-generated REST API)

## AI & LLM Integration
- **LLM Provider:** TBD (OpenAI, Anthropic, or other based on quality/cost analysis)
- **Vector Embeddings:** TBD (aligned with LLM provider choice)
- **Signal Extraction:** LLM-based structured output for note analysis
- **Message Generation:** LLM with context retrieval and rule injection

## Security & Authentication
- **Authentication:** Supabase Auth
- **Secure Storage:** flutter_secure_storage for sensitive on-device data
- **RLS Policies:** Row-level security for all user data tables
- **Secrets Management:** Supabase Edge Function environment variables, Supabase Vault

## Testing & Quality
- **Test Framework:** Flutter Test (unit, widget, integration), pytest (Python tooling)
- **Linting/Formatting:** flutter_lints and dart format (Dart), Black and Flake8 (Python)
- **Monitoring:** Sentry for error tracking across Flutter app and Edge Functions

## Deployment & Infrastructure
- **Mobile Distribution:**
  - iOS: App Store (primary platform for V1)
  - Android: Google Play Store (future phase)
- **CI/CD:** GitHub Actions
- **Database Hosting:** Supabase managed infrastructure
- **CDN/DNS:** Cloudflare (if needed for web assets)

## Third-Party Services
- **Backend Platform:** Supabase (Auth, Database, Storage, Edge Functions)
- **Email:** Supabase transactional email (SMTP) for account-related notifications
- **Push Notifications:** Firebase Cloud Messaging (FCM) via Flutter plugin
- **Error Monitoring:** Sentry

## Platform-Specific
- **iOS:** 
  - Native dictation via in-house Flutter plugin
  - Deep linking for native messaging app integration
  - Contact sync via Flutter contacts plugin
  - Push notifications via APNs (through FCM)
- **Android (Future):**
  - Native speech recognition (future implementation)
  - Intent system for messaging app integration
  - Contact sync via platform APIs
  - Push notifications via FCM

## Development Tools
- **IDE:** VS Code, Android Studio (for Flutter)
- **Version Control:** Git, GitHub
- **Database Management:** Cursor MCP for Supabase, Supabase Dashboard
- **API Testing:** Built-in Supabase API docs, Flutter integration tests

