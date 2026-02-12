# Technology Stack

**Analysis Date:** 2026-02-11

## Languages

**Primary:**
- Rust - Backend services, core system logic, plugins, Tauri desktop app
- TypeScript - Web frontend (React), API clients, tooling
- Go - CLI tools and utility programs

**Secondary:**
- JavaScript - Build scripts, configuration
- HTML/CSS - Web pages, styling via Tailwind CSS

## Runtime

**Environment:**
- Node.js >= 22 (frontend, build tools)
- Rust 2024 edition (backend, desktop)
- Go 1.25.5 (CLI utilities)

**Package Managers:**
- pnpm 10.29.2 - TypeScript/Node packages
- Cargo - Rust packages (workspace with 100+ crates)

## Frameworks

**Core Web:**
- React 19.2.4 - Frontend UI framework
- TanStack React Start 1.159.5 - Full-stack React framework (SSR)
- TanStack React Router 1.159.5 - Client-side routing
- Vite 7.3.1 - Frontend build tool
- Tailwind CSS 4.1.18 - Styling framework
- Drizzle ORM 0.44.7 - Database ORM (TypeScript)

**Backend/API:**
- Axum 0.8 - Rust web framework
- Tauri 2.10 - Desktop application framework
- Restate SDK 0.7.0 - Workflow orchestration runtime
- Tower/Tower-HTTP - HTTP middleware and utilities

**Desktop:**
- Tauri 2.10 - Cross-platform desktop framework (Rust)
- Located at: `apps/desktop/src-tauri/Cargo.toml`

**Testing:**
- Playwright 1.58.2 - E2E browser testing
- @testing-library/react 16.3.2 - React component testing
- Vitest/Jest - Test runners

**Code Quality:**
- ESLint 9.39.2 - JavaScript/TypeScript linting
- Oxlint 1.46.0 - Fast OxC-based linting
- Prettier 3.8.1 with Oxlint plugin - Code formatting
- dprint 0.51.1 - Markdown/TOML formatting
- Turbo 2.8.3 - Monorepo task runner

## Key Dependencies

**Critical Backend:**
- async-stripe 1.0.0-rc.0 - Stripe payment processing
- async-stripe-billing - Billing operations
- tokio 1.x - Async runtime
- reqwest 0.13 - HTTP client
- sentry 0.42.0 - Error tracking
- tracing/tracing-subscriber - Structured logging
- jsonwebtoken 10 - JWT authentication

**Critical Frontend:**
- @tanstack/react-query 5.90.20 - Data fetching/caching
- @tanstack/react-form 1.28.0 - Form state management
- @supabase/supabase-js 2.95.3 - Supabase client
- @stripe/stripe-js 8.7.0 - Stripe frontend SDK
- motion 11.18.2 - Animation library (not framer-motion)
- lucide-react 0.544.0 - Icon library

**Audio Processing:**
- rodio 0.21 - Audio playback
- cpal 0.17 - Audio I/O
- dasp 0.11.0 - DSP utilities
- realfft 3.5.0 - FFT operations
- hound 3.5.1 - WAV file handling
- deepgram 0.7 - Speech-to-text SDK

**Database & Storage:**
- libsql 0.9.24 - SQLite client
- sqlx 0.8 - SQL toolkit (feature-gated)
- drizzle-orm 0.44.7 - ORM for web frontend
- aws-sdk-s3 1.66.0 - S3 storage

**LLM/AI:**
- async-openai - OpenAI API client (custom fork)
- openrouter - OpenRouter API integration
- deepgram - Speech recognition

**macOS Integration:**
- objc2 0.6 - Objective-C interop
- objc2-app-kit 0.3.2 - AppKit bindings
- cidre 0.14 - Core Data access
- swift-rs - Swift interop

**Workspace Internal:**
- ~100 internal crates prefixed with `hypr-*` for modularity
- Located at: `crates/`, `plugins/`, `packages/`

## Configuration

**Environment:**
- `.env.supabase` - Supabase configuration (referenced in scripts)
- Environment variable loading via `envy` crate (Rust) and `dotenvx`
- Required env vars configured per service module

**Build:**
- `Cargo.toml` (root workspace) - 100+ crate configuration
- `tsconfig.json` - TypeScript compilation
- `vite.config.ts` - Vite build configuration
- `tailwind.config.js` - Tailwind CSS customization
- `eslint.config.js` - Linting rules
- `.prettierrc` - Prettier formatting (80 char line width, 2 space indent)
- `dprint.json` - dprint configuration
- `bacon.toml` - Cargo build watcher config
- `.oxlintrc.json` - Oxlint configuration

**Feature Flags:**
- Desktop: `devtools`, `automation`, `apple-calendar-fixture`
- Located in: `apps/desktop/src-tauri/Cargo.toml`

## Platform Requirements

**Development:**
- macOS, Linux, or Windows (Tauri supports all three)
- Rust toolchain 2024 edition
- Node.js >= 22
- Go 1.25.5
- Xcode Command Line Tools (macOS)

**Production:**
- Desktop: Packaged Tauri app for macOS, Linux, Windows
- Web: Deployed via Netlify or similar (TanStack Start)
- API: Docker container or cloud deployment (Axum/Tokio)
- Database: PostgreSQL (via Supabase) or Turso (SQLite)

**CI/CD:**
- GitHub Actions (40+ workflow files in `.github/workflows/`)
- Multiple pipelines for: API, Desktop, Web, E2E, Publishing
- See: `api_ci.yaml`, `desktop_ci.yaml`, `desktop_cd.yaml`, `web_ci.yaml`

---

*Stack analysis: 2026-02-11*
