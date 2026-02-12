# Architecture

**Analysis Date:** 2026-02-11

## Pattern Overview

**Overall:** Distributed monorepo with modular component-based architecture across Rust backend services and TypeScript/React frontend applications, using workspace-driven dependency management and plugin-based extensibility.

**Key Characteristics:**
- Workspace-driven modular Rust architecture with 100+ crates and plugin system
- Layered backend with API gateway (Axum-based) that proxies to specialized services
- React Server Components (TanStack Start) frontend with file-based routing
- Desktop application as Tauri wrapper around shared business logic
- Micro-crate architecture enabling code reuse across apps (API, Desktop, Bot)
- Plugin system for Tauri desktop extending core functionality at runtime

## Layers

**API Gateway (Backend):**
- Purpose: HTTP entry point, authentication, request routing, observability integration
- Location: `apps/api/src/main.rs`
- Contains: Router setup, middleware chains (CORS, Tracing, Sentry), authentication state, service composition
- Depends on: Authentication crates, LLM proxy, STT proxy, subscription service, support service, Nango integration
- Used by: Web app, Desktop app (via HTTP), Bot (Slack)

**Service Modules (Backend):**
- Purpose: Domain-specific business logic isolated in workspace crates
- Examples: `crates/api-subscription/`, `crates/api-calendar/`, `crates/api-support/`, `crates/api-nango/`
- Pattern: Each exposes `router()` async function returning `axum::Router`, takes config struct in constructor
- Depends on: Database clients, external API clients, shared utilities
- Used by: API gateway routes them via `.nest()` and `.merge()`

**LLM & STT Proxy Services:**
- Purpose: Abstract multiple LLM/STT providers behind unified interface
- Location: `crates/llm-proxy/src/`, `crates/transcribe-proxy/src/`
- Pattern: Proxy pattern with provider abstraction, analytics integration
- Router methods: `listen_router()` for streaming, `chat_completions_router()` for standard completion
- Configuration: Passed as `LlmProxyConfig`/`SttProxyConfig` with env-based provider selection

**Desktop Application (Tauri + Rust):**
- Purpose: Native desktop wrapper with background services and plugin system
- Location: `apps/desktop/src-tauri/src/lib.rs`
- Contains: Plugin initialization, Tauri command mapping, actor supervisor setup
- Key modules: `supervisor.rs` (actor system), `agents.rs` (background tasks), `commands.rs` (IPC handlers), `store.rs` (state), `control.rs`, `ext.rs`
- Pattern: Actor-based concurrency using `ractor` crate with dynamic supervisor spawning child actors

**Web Application (React):**
- Purpose: Public-facing web interface and documentation
- Location: `apps/web/src/`
- Framework: TanStack Start (React SSR), TanStack Router (file-based routing), React Query
- Entry point: `src/router.tsx` - router setup with QueryClient, Sentry, PostHog, provider wrapping
- Root layout: `src/routes/__root.tsx` - metadata, scripts, root context setup

**Web Layer Structure:**
- Routes: `src/routes/` - TanStack Router file-based routes (auto-generated tree in `routeTree.gen.ts`)
- Components: `src/components/` - React component library (layout, UI, admin components)
- Hooks: `src/hooks/` - Custom React hooks for shared stateful logic
- Functions: `src/functions/` - Server/client async functions (API calls, data fetching)
- Middleware: `src/middleware/` - Supabase SSR, CORS, Drizzle, Nango middleware setup
- Providers: `src/providers/` - Context providers (PostHog analytics)
- Utils: `src/utils/` - Helper functions (download, sitemap generation)

**Plugin System (Tauri):**
- Purpose: Runtime extensibility for desktop app features
- Location: `plugins/` - 50+ plugin packages
- Pattern: Each plugin is standalone Rust + TypeScript combo (`plugins/[name]/` with `src-tauri/src/` and `dist/`)
- Plugins initialized in `apps/desktop/src-tauri/src/lib.rs` via `.plugin()` builder chain
- Examples: `plugins/auth/`, `plugins/db2/`, `plugins/listener/`, `plugins/local-llm/`, `plugins/updater2/`

**Crate Library (Shared):**
- Purpose: Reusable domain logic and utilities across workspace
- Location: `crates/` - 100+ crates with specific responsibilities
- Organization: Named by domain (e.g., `api-*` for API modules, `transcribe-*` for STT providers, `audio-*` for audio processing)
- Pattern: Each crate is a Cargo package exposed via workspace root `Cargo.toml` with `hypr-*` naming convention

## Data Flow

**Web App Request Flow:**

1. User interaction in React component (`src/components/` or `src/routes/`)
2. Component uses React Query hook or React Form hook to trigger async function
3. Async function in `src/functions/` makes HTTP request to API (`api/url`)
4. API request goes to Axum router in `apps/api/src/main.rs`
5. Router middleware adds tracing/observability, validates auth via Supabase JWT
6. Matched route handler in service module (e.g., `hypr-api-subscription`) executes business logic
7. Service may call external APIs (Stripe, Nango, database) or internal crates
8. Response returned through middleware layers (error handling, Sentry reporting)
9. React component receives response, updates state, re-renders via React Query cache invalidation

**Desktop App Flow:**

1. User interaction in Tauri window (WebView)
2. Tauri command invoked via `invoke()` from TypeScript frontend
3. Command handler in `apps/desktop/src-tauri/src/commands.rs` receives request
4. Handler may spawn actor task via supervisor or call plugin directly
5. Background actor (via `ractor` supervisor) processes work asynchronously
6. Result stored in app state (`store.rs`) or sent back via IPC event
7. Frontend listens via `tauri://events` and updates UI

**State Management:**

**Backend:**
- HTTP request/response model - stateless across requests
- Per-request state passed via middleware context (auth, analytics)
- Config structs injected at service creation: `LlmProxyConfig`, `SubscriptionConfig`, etc.
- Sentry integration for error reporting at top-level middleware

**Frontend:**
- React Query for server state caching and synchronization
- React Form (`@tanstack/react-form`) for form state management (no manual error state)
- React Context via providers (`PostHogProvider`, `MaybeOutlitProvider`)
- Router state managed by TanStack Router
- No manual state management - delegated to React Query and form libraries

**Async Patterns:**

**Backend:**
- Axum async handlers with `.await` patterns
- Middleware chains evaluated left-to-right, applied as tower layers
- Streaming responses via SSE (server-sent events) for LLM/STT endpoints (`listen_router()`)
- Sentry/Tracing at middleware level for automatic instrumentation

**Frontend:**
- React Query mutations for mutations with automatic cache invalidation
- React Query queries for data fetching with built-in refetch/retry logic
- Async server functions in TanStack Start (functions in `src/functions/`)
- Middleware for cross-cutting concerns (auth, CORS, data transformation)

## Key Abstractions

**Service Abstraction (Backend):**
- Purpose: Encapsulate domain logic with consistent router interface
- Pattern: Each service module exposes `pub async fn router(config: ServiceConfig) -> Router`
- Examples:
  - `hypr-api-subscription::router(config)` - subscription/billing operations
  - `hypr-api-calendar::router(config)` - calendar integrations via Nango
  - `hypr-api-support::router(config)` - support ticket management
- Benefit: Services pluggable, testable, independently deployable

**Proxy Abstraction (LLM/STT):**
- Purpose: Provide unified interface to multiple providers
- Implementation: `hypr-llm-proxy`, `hypr-transcribe-proxy`
- Methods:
  - `chat_completions_router()` - standard request/response
  - `listen_router()` - streaming responses (server-sent events)
- Config holds provider selection logic (`LlmProxyConfig`, `SttProxyConfig`)
- Allows swapping providers without service code changes

**Plugin Architecture (Desktop):**
- Purpose: Extend desktop app with new capabilities at runtime
- Pattern: Plugin implements Tauri command handlers
- Lifecycle: Initialized via `.plugin()` in Tauri builder, registered in IPC handler registry
- Communication: TypeScript `invoke()` → Rust command handler → plugin logic
- Examples: `tauri_plugin_auth`, `tauri_plugin_db2`, `tauri_plugin_local_llm`

**Actor Model (Desktop Background):**
- Purpose: Manage concurrent background operations reliably
- Implementation: `ractor` crate with `DynamicSupervisor`
- Root: `RootSupervisorContext` spawned in `supervisor.rs`, manages child actors
- Pattern: Actors spawn via supervisor, receive messages, process independently
- Failure handling: Supervisor restarts crashed actors up to max_restarts
- Benefit: Isolation, automatic restart on crash, decoupled async work

**Authentication State (Backend):**
- Purpose: Provide auth context to route handlers
- Implementation: `AuthState` struct wrapping Supabase URL
- Variants:
  - `auth_state_pro` - with required entitlement check
  - `auth_state_basic` - standard JWT validation
  - `auth_state_support` - optional auth for support endpoints
- Usage: Applied as middleware with `require_auth` or `optional_auth`

## Entry Points

**API Server:**
- Location: `apps/api/src/main.rs`
- Triggers: Binary execution via `cargo run` or Docker container
- Responsibilities:
  - Environment variable parsing via `env()` function
  - Service configuration creation (Auth, LLM, STT, Calendar, Subscription, Support)
  - Router composition with nested services
  - Middleware stack setup (CORS, Tracing, Sentry)
  - HTTP server bind and listen on `[::]:8000` (default)

**Desktop Application:**
- Location: `apps/desktop/src-tauri/src/lib.rs::main()`
- Triggers: Tauri build process, executed as app binary
- Responsibilities:
  - Tokio runtime initialization
  - Root supervisor actor spawn for background operations
  - Sentry client initialization with error handling
  - Plugin system initialization via builder chain
  - Tauri app window creation and IPC setup
  - Permission system setup

**Web Application:**
- Location: `apps/web/src/router.tsx::getRouter()`
- Triggers: Vite dev server or SSR build
- Responsibilities:
  - QueryClient creation for React Query
  - Router tree creation from auto-generated `routeTree.gen.ts`
  - Context setup with providers (PostHog, Outlit, QueryClient)
  - Sentry SDK initialization with browser tracing
  - Router SSR query integration setup

**Desktop Frontend (Tauri WebView):**
- Location: `apps/desktop/src/[entry]` (Svelte/React based on implementation)
- Triggers: Tauri window creation
- Responsibilities: Invoke Tauri commands, listen to events, render UI

## Error Handling

**Strategy:** Middleware-based error capture with Sentry integration, typed error responses at service layer, React Query automatic error state in frontend

**Patterns:**

**Backend:**
- Sentry middleware at router level captures all panics and errors: `NewSentryLayer::<Request<Body>>::new_from_top()`
- Tracing middleware provides structured logging context: `TraceLayer::new_for_http()`
- Service-layer errors return Axum response types (Result<Json<T>, ApiError>)
- Error types implement `IntoResponse` for automatic HTTP status mapping
- Example: `AuthState::require_auth` middleware returns 401 on token validation failure

**Frontend:**
- React Query provides automatic error state: `useQuery().error` and `useMutation().error`
- Error handling in components checks `error` field without manual try/catch
- Form validation via React Form with `onBlur` and `onChange` error fields
- Sentry SDK captures unhandled errors: `@sentry/tanstackstart-react`
- PostHog event tracking integrates with error reporting

**Observability:**

**Backend:**
- Tracing: Structured logging via `tracing_subscriber`, span context propagation
- Errors: Sentry captures all server errors with request context, breadcrumbs
- Analytics: `AnalyticsClientBuilder` provides PostHog analytics for feature tracking
- Health check: `GET /health` endpoint skipped in trace logging to reduce noise

**Frontend:**
- Error tracking: Sentry captures client errors with breadcrumbs and replay data
- Analytics: PostHog tracks user interactions, feature usage
- Performance: Sentry performance monitoring via TanStack Router integration

## Cross-Cutting Concerns

**Logging:** Structured tracing via `tracing` crate (backend), console/Sentry (frontend); request/response bodies excluded by default for security

**Validation:**
- Backend: Service modules validate input before business logic (Serde deserialization provides basic validation)
- Frontend: React Form provides client-side validation; server-side errors returned via mutation error state

**Authentication:**
- Backend: JWT validation via `AuthState` middleware extracting Supabase JWT from Authorization header
- Frontend: Supabase SSR context via `@supabase/ssr` middleware providing auth state to components
- Desktop: Plugin-based auth system via `tauri_plugin_auth`

**Authorization:**
- Backend: Entitlement checks in `AuthState` (e.g., `with_required_entitlement("hyprnote_pro")`)
- Frontend: Route-level access control via layout routes checking auth context
- Desktop: Permission system via `tauri_plugin_permissions` with manifest declarations

---

*Architecture analysis: 2026-02-11*
