# External Integrations

**Analysis Date:** 2026-02-11

## APIs & External Services

**Payment Processing:**
- Stripe - Payment handling, subscriptions, billing
  - SDK: `async-stripe` (v1.0.0-rc.0), `stripe` (Node)
  - Auth: `STRIPE_SECRET_KEY`
  - Client: `crates/api-subscription/src/state.rs` manages Stripe client
  - Pricing: `STRIPE_MONTHLY_PRICE_ID`, `STRIPE_YEARLY_PRICE_ID`
  - Frontend: `apps/web/src/functions/stripe.ts`

**Integration Platform:**
- Nango - OAuth and API integrations
  - SDK: `nango` crate at `crates/nango/`
  - Auth: `NANGO_API_KEY`
  - Optional: `NANGO_API_BASE` (custom endpoint)
  - Config: `crates/api-env/src/lib.rs` defines NangoEnv
  - Frontend middleware: `apps/web/src/middleware/nango.ts`

**Analytics:**
- PostHog - Product analytics
  - SDK: `@posthog/react` (v1.7.1), `posthog-js` (v1.345.3)
  - Crate: `crates/posthog/`
  - Usage: `apps/web/src/providers/posthog.tsx`
  - Agent support: `packages/agent-support/src/tools/posthog.ts`

**Error Tracking:**
- Sentry - Error tracking and monitoring
  - SDK: `sentry` (v0.42.0), `@sentry/tanstackstart-react` (v10.38.0)
  - Configuration: Rust crates use `sentry` with tower/tower-axum middleware
  - Desktop plugin: `tauri-plugin-sentry`

**AI & LLM:**
- OpenAI - Speech-to-text (Whisper), language models
  - SDK: `async-openai` (custom fork from fastrepl)
  - Realtime API support enabled
  - Transcription: `crates/transcribe-openai/`
  - Config: `OPENROUTER_API_KEY` for OpenRouter fallback

- OpenRouter - LLM routing and models
  - SDK: `crates/openrouter/`
  - Auth: `OPENROUTER_API_KEY`
  - Proxy: `crates/llm-proxy/`

- Deepgram - Speech recognition
  - SDK: `deepgram` (v0.7)
  - Transcription: `crates/transcribe-deepgram/`
  - Features: `["listen"]` enabled

- AWS Bedrock - ML inference
  - Plugin: `tauri-plugin-bedrock`
  - Crate: `crates/llm-proxy/` may delegate to Bedrock

**Cloud Providers:**

*AWS:*
- S3 Storage
  - SDK: `aws-sdk-s3` (v1.66.0), `aws-config` (v1.5.11)
  - Crate: `crates/s3/`
  - Features: Latest behavior version
  - Credentials: `aws-credential-types` with hardcoded support

- Transcription (AWS)
  - Crate: `crates/transcribe-aws/`

*Google Cloud:*
- GCP Transcription
  - Crate: `crates/transcribe-gcp/`

*Azure:*
- Azure Transcription
  - Crate: `crates/transcribe-azure/`

**Search & Knowledge:**
- Exa (ex-perplexity) - Search API
  - SDK: `exa-js` (v1.10.2)
  - Usage: `apps/web/src/routes/api/` for search operations

**Calendar & Contacts:**
- Google Calendar
  - Crate: `crates/google-calendar/`

- Apple Calendar/Contacts
  - Plugins: `tauri-plugin-apple-calendar`, `tauri-plugin-apple-contact`
  - Crates: `crates/apple-calendar/` placeholder

## Data Storage

**Databases:**

*PostgreSQL (Supabase):*
- Provider: Supabase (managed PostgreSQL)
- Connection: Environment variables
  - `SUPABASE_URL` - API endpoint
  - `SUPABASE_ANON_KEY` - Public anonymous key
- Clients:
  - Frontend: `@supabase/supabase-js` (v2.95.3), `@supabase/ssr`
  - Backend: Custom Supabase client via reqwest
  - Auth: `crates/supabase-auth/` handles JWT validation
- Usage:
  - Frontend middleware: `apps/web/src/middleware/supabase.ts`
  - User authentication and data storage
  - Real-time subscriptions (via Supabase)

*SQLite/Turso:*
- Turso - Managed SQLite over HTTP
  - SDK: `crates/turso/` (cached HTTP client)
  - ORM: Drizzle ORM for web frontend
  - Local SQLite option available

**File Storage:**
- AWS S3 - Cloud file storage
  - SDK: `aws-sdk-s3`
  - Implementation: `crates/s3/`
  - Credentials: AWS config/environment
- Local filesystem - Via Tauri `plugin-fs2` for desktop
- Supabase Storage - File uploads via Supabase
  - Frontend: `apps/web/src/functions/supabase-media.ts`

**Caching:**
- In-memory caching via `cached` crate (v0.55.1)
- Turso uses HTTP caching layer
- TanStack Query caching on frontend

## Authentication & Identity

**Auth Provider:**
- Supabase Auth - Custom implementation
  - Backend: JWT token validation via `crates/supabase-auth/`
  - Frontend: Session management via Supabase SDK
  - Middleware: Auth guards in `apps/web/src/middleware/`
  - Claims validation: `crates/supabase-auth/src/claims.rs`

**JWT Handling:**
- Crate: `jsonwebtoken` (v10) with `rust_crypto` backend
- Used in: API authentication, token validation

## Monitoring & Observability

**Error Tracking:**
- Sentry - Production error monitoring
  - Init config: `apps/desktop/` via plugin
  - Backend integration: `axum` middleware in API
  - Frontend integration: `@sentry/tanstackstart-react`

**Logs:**
- Structured logging via `tracing` crate
- Subscriber: `tracing-subscriber` with env-filter
- Desktop app: `tauri-plugin-tracing`
- API uses tower middleware for request tracing

**Metrics & Analytics:**
- PostHog - Product analytics
- Openstatus - Status page monitoring
  - Crate: `crates/openstatus/`

## CI/CD & Deployment

**Hosting:**
- Netlify - Web frontend deployment (TanStack Start support)
  - Build: Vite with `@netlify/vite-plugin-tanstack-start`
  - Caching: `@netlify/cache` integration

- Custom - API deployment (Axum/Tokio apps)

- Self-hosted - Desktop app (Tauri)

**CI Pipeline:**
- GitHub Actions - Monorepo CI/CD
  - Primary workflow: `.github/workflows/ci.yaml`
  - Per-app workflows:
    - API: `api_ci.yaml`, `api_cd.yaml`
    - Desktop: `desktop_ci.yaml`, `desktop_cd.yaml`, `desktop_e2e.yaml`
    - Bot: `bot_ci.yaml`, `bot_cd.yaml`
  - Desktop publishing: `desktop_publish.yaml`
  - E2E testing: `desktop_e2e.yaml`

**Build Tools:**
- Turbo - Monorepo task orchestration
- Cargo - Rust workspace builds
- Vite - Frontend dev server and builds

## Environment Configuration

**Required Environment Variables (Web Frontend):**
- `SUPABASE_URL` - Supabase API endpoint
- `SUPABASE_ANON_KEY` - Supabase public key
- `STRIPE_PUBLIC_KEY` - Stripe publishable key (frontend)
- `STRIPE_SECRET_KEY` - Stripe secret key (backend only)
- `VITE_APP_URL` - Frontend URL (dev)
- `NANGO_API_KEY` - Nango API key
- `OPENROUTER_API_KEY` - OpenRouter API key
- `.env.supabase` - Sourced in dev scripts

**Secret Storage:**
- Development: `.env` files (loaded via dotenvx)
- Production: Environment variables via deployment platform
- Stripe keys, API keys, and auth tokens stored as secrets

## Webhooks & Callbacks

**Incoming Webhooks:**
- Stripe webhooks for payment events
  - Endpoint: TBD in API routes
  - Events: payment_intent.succeeded, customer.subscription.updated, etc.

- Nango webhooks for integration state changes
  - Configured via Nango dashboard

**Outgoing:**
- Webhook plugin: `tauri-plugin-webhook`
- Custom webhook system for desktop app events

## Version Compatibility Notes

**Patch Dependencies:**
- `async-openai` - Custom fork from fastrepl (git-based)
  - Rev: 6404d307f3f706e818ad91544dc82fac5c545aee
  - Features: Realtime API support
- `gbnf-validator` - Custom fork from fastrepl
  - Rev: 3dec055
- `silero-rs` - Custom fork from emotechlab
  - Rev: 26a6460
  - Package: silero

**Toolchain:**
- Rust Edition: 2024
- MSVC link fix for esaxx-rs (patched)

---

*Integration audit: 2026-02-11*
