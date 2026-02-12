# Codebase Concerns

**Analysis Date:** 2026-02-11

## Tech Debt

**Monolithic Components with Multiple Concerns:**
- Issue: Several large UI components handle too many responsibilities (state management, API calls, rendering, editing logic)
- Files: `apps/web/src/routes/admin/collections/index.tsx` (3,628 lines), `apps/web/src/routes/admin/media/index.tsx` (2,526 lines), `apps/desktop/src/components/main/body/index.tsx` (1,031 lines)
- Impact: High maintenance burden, difficult to test, increased bug surface area, poor code reusability, slow refactoring
- Fix approach: Extract logical concerns into smaller, focused components. Separate API communication into custom hooks. Extract state management into dedicated stores. Create shared utilities for common patterns.

**Manual Form State Management:**
- Issue: Forms use multiple `useState` hooks for error tracking and input state instead of leveraging existing patterns (useForm from tanstack-form)
- Files: `apps/web/src/routes/admin/collections/index.tsx` (77 useState/useRef/useCallback calls), various form components
- Impact: Repetitive code, inconsistent error handling, difficult to maintain validation logic, susceptible to race conditions
- Fix approach: Migrate all forms to use useForm from @tanstack/form with proper validation schemas. Use tanstack-query mutations for server interactions.

**Hardcoded User Mappings:**
- Issue: GitHub username to author mapping is hardcoded in application code
- Files: `apps/web/src/functions/github-content.ts` (lines 28-34)
- Impact: Not scalable, requires code changes to add new team members, difficult to maintain as team grows
- Fix approach: Move author mappings to database or configuration file with admin interface to manage mappings dynamically

**Test Coverage Gaps:**
- Issue: Only 51 test files for 937 source files (~5% coverage ratio), major functional areas untested
- Files: Missing tests for: admin content management, media operations, complex state synchronization, API error handling, auth flows
- Impact: Regressions undetected until production, refactoring risky, confidence in changes low
- Priority: High
- Fix approach: Focus on critical paths first - admin operations, auth flows, API integrations. Establish minimum coverage requirements for new code. Create shared test utilities and fixtures.

## Known Bugs

**Auto-Save Synchronization Issues:**
- Symptoms: Unsaved changes may not persist when tab is closed, summary data can be lost during navigation
- Files: `apps/web/src/routes/admin/collections/index.tsx` (autosave logic around line 300), multiple instances of tab management
- Trigger: Close editor tab while autosave is in flight, navigate away during save operation
- Workaround: Manually trigger save before closing tab
- Root cause: State updates not coordinated with onComplete callbacks, race condition between tab unmount and API calls

**Event Listener Memory Leaks:**
- Symptoms: Application memory usage increases over time, potential performance degradation
- Files: `apps/desktop/src/components/settings/general/notification.tsx`, `apps/desktop/src/components/chat/trigger.tsx`, multiple setTimeout calls without cleanup
- Trigger: Navigate between components with event listeners, leave app running for extended periods
- Current pattern: Some components properly clean up (e.g., `useEffect(() => { ... return () => removeEventListener() })`), but others don't consistently implement cleanup

**Pending Query State Leaks:**
- Symptoms: Queries remain in loading/pending state indefinitely when API errors occur
- Files: `apps/web/src/routes/admin/collections/index.tsx` (pendingPR, branchFile queries)
- Trigger: Network timeout or API error during file fetch
- Impact: UI remains in loading state, user cannot interact with interface
- Current issue: Error handling doesn't properly reset loading states in all error paths

## Security Considerations

**GitHub Token Management:**
- Risk: GitHub personal access tokens stored in Supabase auth_admin table accessible through database queries
- Files: `apps/web/src/functions/github-content.ts` (lines 76-90)
- Current mitigation: Supabase RLS policies, auth requirement checks
- Recommendations:
  - Implement token rotation strategy
  - Use OAuth apps instead of PATs where possible
  - Add audit logging for token usage
  - Consider token expiration and refresh mechanisms
  - Encrypt tokens at rest in database

**API Error Messages Exposing Internal Details:**
- Risk: Generic error messages could expose implementation details to client
- Files: Various API endpoints returning error objects with internal messages
- Current mitigation: Error messages are passed through but not logged comprehensively
- Recommendations:
  - Implement error code system (E001, E002, etc.) with sanitized user messages
  - Log full error details server-side only
  - Return generic client-facing messages to prevent information leakage

**Missing Input Validation:**
- Risk: User inputs not validated consistently before API submission
- Files: `apps/web/src/routes/admin/collections/index.tsx` (filename sanitization, folder validation)
- Current mitigation: Basic sanitization with `sanitizeFilename()` function, folder whitelist (`VALID_FOLDERS`)
- Recommendations:
  - Implement comprehensive Zod schemas for all user inputs
  - Validate folder paths against whitelist more strictly
  - Add maximum length validation for text inputs
  - Implement rate limiting on API endpoints

**Insufficient Auth Checks:**
- Risk: Admin endpoints assume authenticated user without comprehensive permission validation
- Files: `apps/web/src/routes/admin/collections/index.tsx` (beforeLoad auth check only)
- Current mitigation: `beforeLoad` auth checks that redirect to GitHub auth
- Recommendations:
  - Add role-based access control (RBAC) to admin endpoints
  - Implement granular permission checks (can edit article, can delete content, etc.)
  - Add audit logging for all admin operations
  - Consider implementing admin approval workflows for sensitive operations

**Supabase Query Vulnerabilities:**
- Risk: `.single()` on queries without proper error handling could fail silently
- Files: `apps/web/src/functions/github-content.ts` (line 80)
- Impact: Query returning null instead of user record could bypass auth checks
- Recommendations:
  - Always handle potential null returns from `.single()`
  - Use strict equality checks for user IDs
  - Add logging for unexpected query results

## Performance Bottlenecks

**Inefficient Query Invalidation:**
- Problem: Broad query invalidation patterns could refetch entire data sets unnecessarily
- Files: `apps/web/src/routes/admin/collections/index.tsx` (line 300+), `apps/web/src/routes/admin/media/index.tsx`
- Example: `queryClient.invalidateQueries({ queryKey: ["mediaItems"] })` invalidates all media queries regardless of affected scope
- Impact: Unnecessary network requests, increased server load, slower UI updates
- Improvement path: Use specific query invalidation based on changed data. Implement optimistic updates instead of refetching. Use query-level filters to minimize invalidation scope.

**Large Component Re-renders:**
- Problem: Complex components like collections/index.tsx have many render dependencies that could cause unnecessary re-renders
- Files: `apps/web/src/routes/admin/collections/index.tsx`, `apps/desktop/src/components/main/body/index.tsx`
- Impact: Frame drops, sluggish UI, higher CPU usage
- Current metrics: Collections component has 77+ state variables and dependencies
- Improvement path: Break into smaller memoized components. Use `useCallback` more consistently. Consider extracting tab logic into separate component. Implement virtual scrolling for large lists.

**Query Polling with Long Intervals:**
- Problem: Some queries use staleTime/gcTime without polling strategy
- Files: `apps/web/src/routes/admin/collections/index.tsx` (staleTime: 30000-60000)
- Impact: Stale data displayed to users, potential inconsistency between local state and server
- Improvement path: Implement aggressive polling for frequently-changed data (< 10s). Use background sync for critical updates. Consider WebSocket subscriptions for real-time data.

**No Pagination/Virtualization:**
- Problem: Large lists (articles, media) loaded entirely into memory without pagination or virtual scrolling
- Files: `apps/web/src/routes/admin/media/index.tsx`, collection browsing
- Impact: High memory usage with large datasets, slow initial load, poor mobile performance
- Improvement path: Implement infinite scroll with cursor pagination. Use react-virtual for virtualization. Load items on-demand.

## Fragile Areas

**Tab Management System:**
- Files: `apps/desktop/src/store/zustand/tabs` (referenced throughout)
- Why fragile: Multiple components depend on tab state synchronization. Tab state changes can cascade. Race conditions possible during tab switching/closing.
- Safe modification:
  - Always use `useShallow` for state subscriptions to prevent unnecessary re-renders
  - Batch tab updates together
  - Add invariant checks for tab state consistency
  - Test tab lifecycle scenarios (open/close/switch/duplicate)
- Test coverage: Minimal - tabs used extensively but not tested comprehensively

**Editor State Synchronization:**
- Files: `apps/web/src/routes/admin/collections/index.tsx` (EditorData, FileContent, ArticleMetadata interfaces)
- Why fragile: Multiple state sources (branchFile, pendingPRFile, local editor state, autosave state). Data flow between sources unclear. Multiple mutations can race.
- Safe modification:
  - Document state flow explicitly
  - Create single source of truth for editor data
  - Use optimistic updates with proper rollback
  - Implement transaction-like semantics for compound operations
- Test coverage: No tests for complex editor interactions

**Calendar Sync Timeout Management:**
- Files: `apps/desktop/src/components/settings/calendar/configure/apple/context.tsx` (lines 35-80)
- Why fragile: Uses `toggleSyncTimeoutRef` with multiple state variables (pendingTaskRunId, isDebouncing). Cleanup in useEffect could race with new sync requests.
- Safe modification:
  - Centralize timeout/debounce logic
  - Use AbortController for cancellation
  - Add state invariant assertions
  - Test rapid toggle scenarios
- Current concern: Line 60 calls `scheduleEventSync()` in cleanup - could trigger unwanted syncs on unmount

**Extension Registry Loading:**
- Files: `apps/desktop/src/components/main/body/index.tsx` (line 52, loadExtensionPanels)
- Why fragile: Extension loading happens once on mount with `void loadExtensionPanels()`. If extensions modify global state, race conditions possible. No error handling shown.
- Safe modification:
  - Add try-catch and error boundary
  - Implement extension loading strategy (sequential vs parallel)
  - Add validation for extension manifest
  - Test extension load failures

## Scaling Limits

**Single Monolithic Supabase Connection:**
- Current capacity: Single Supabase project handling all auth, user data, admin data
- Limit: Will hit connection limits and API rate limits as user base grows
- Impact: Auth failures during traffic spikes, slow queries affecting all users
- Scaling path:
  - Implement read replicas for Supabase
  - Add connection pooling
  - Separate admin operations to dedicated service
  - Implement API rate limiting and backpressure
  - Consider multi-region deployment

**Desktop App Data Synchronization:**
- Current capacity: In-process TinyBase store with single-device scope
- Limit: Multiple device support, cloud sync, conflict resolution undefined
- Scaling path:
  - Implement cloud sync protocol (CRDTs for conflict resolution)
  - Add device identification and sync orchestration
  - Test multi-device consistency scenarios
  - Consider P2P sync for offline-first capabilities

**Media Storage Without Limits:**
- Current capacity: Media uploaded to storage without quotas or cleanup
- Limit: Storage costs will grow linearly with users, no retention policies
- Scaling path:
  - Implement user storage quotas
  - Add media expiration/cleanup policies
  - Implement tiered storage (hot/cold)
  - Add compression for media files

## Dependencies at Risk

**Outdated Type Definitions:**
- Risk: `apps/desktop/src/types/tauri.gen.ts` (21 instances of `any`) and `apps/web/src/routeTree.gen.ts` (190 instances of `any`)
- Impact: Type safety reduced, potential runtime errors not caught
- These are generated files but indicate underlying type generation issues
- Migration plan: Review type generation processes, ensure generated types are fully typed

**TanStack Query Usage Patterns:**
- Risk: Inconsistent query configuration (staleTime varies: 30000-60000ms), no global query client configuration documented
- Impact: Unpredictable cache behavior, difficult to reason about data freshness
- Migration plan: Establish global default query configuration. Document staleTime decisions. Implement query factory functions.

**Media API Integration Issues:**
- Risk: `apps/web/src/hooks/use-media-api.tsx` has 7 fetch calls but incomplete error handling
- Impact: Failed media operations not user-facing, uploads could fail silently
- Migration plan: Implement comprehensive error handling with user feedback

## Missing Critical Features

**No Offline Support:**
- Problem: Web app requires connectivity; desktop app has no sync strategy for offline work
- Blocks: Mobile-friendly note-taking, reliable experience on unreliable connections
- Impact: Users cannot take notes during network interruptions

**No Conflict Resolution:**
- Problem: No CRDT or conflict resolution for simultaneous edits across devices/sessions
- Blocks: Multi-device seamless experience, collaborative editing
- Impact: Last-write-wins could lose data, no merge strategies

**No Data Encryption:**
- Problem: User data stored in plain text in Supabase
- Blocks: End-to-end encryption feature, HIPAA/data protection compliance
- Impact: Data privacy concerns, regulatory compliance gaps

**No Audit Logging:**
- Problem: No comprehensive logging of admin operations, content changes, access patterns
- Blocks: Security incident investigation, regulatory compliance, accountability
- Impact: Cannot track who changed what or when, no forensic data

## Test Coverage Gaps

**Admin Operations Untested:**
- What's not tested: Content creation, editing, deletion, publishing workflows, PR management, branch operations
- Files: `apps/web/src/routes/admin/collections/index.tsx`, `apps/web/src/routes/admin/media/index.tsx`
- Risk: Critical business logic could have regressions undetected
- Priority: High - these are revenue-impacting features

**API Error Handling Untested:**
- What's not tested: Network timeouts, rate limiting, server errors, malformed responses
- Files: All fetch-based API calls throughout apps/web, apps/api
- Risk: Error states could cascade into UI failures or data loss
- Priority: High - affects reliability

**State Synchronization Untested:**
- What's not tested: Tab switching, component unmounting during async operations, race conditions in mutations
- Files: Tab management, editor state, form submissions
- Risk: Race conditions and state corruption in edge cases
- Priority: High - user-facing data integrity

**Desktop App Integrations Untested:**
- What's not tested: Tauri IPC calls, native integrations (calendar, audio), platform-specific behavior
- Files: `apps/desktop/src/` components interacting with native APIs
- Risk: Silent failures on specific platforms, unexpected behavior in edge cases
- Priority: Medium - affects platform stability

---

*Concerns audit: 2026-02-11*
