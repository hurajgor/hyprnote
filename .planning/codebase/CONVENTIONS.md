# Coding Conventions

**Analysis Date:** 2026-02-11

## Naming Patterns

**Files:**
- Kebab-case: `use-form.ts`, `day-event.tsx`, `calendar-day.tsx`, `day-session.tsx`
- Index files for exports: `index.ts` for re-exporting modules
- Test files: `.test.ts` or `.spec.ts` suffix (colocated with source)
- No type-specific suffixes (types are colocated in the same file)

**Functions:**
- camelCase for all functions: `extractKeywordsFromMarkdown`, `buildTimelineBuckets`, `getBucketInfo`
- React hooks: `use` prefix following standard convention: `useAutoEnhance`, `usePermissions`, `useKeywords`
- Private/internal functions: lowercase with underscores: `removeCodeBlocks`, `extractHashtags`, `stripMarkdownFormatting`
- Factory/creator functions: `create` prefix: `createOrganizationPersister`, `createServerFn`, `createSpecialist`
- Non-exported arrow functions allowed: `const combineKeywords = (items) => { ... }`

**Variables:**
- camelCase for all variables: `linkedSessionId`, `rawMd`, `titleCase`, `matchedKeywords`
- Constants in files: still camelCase (not UPPER_CASE): `MergeableStoreOnly`, `SYSTEM_TIME`, `MOCK_DATA_DIR`
- Boolean prefixes: `is*`, `has*`, `can*`: `isCurrentMonth`, `isFirstColumn`, `isSameDay`

**Types:**
- PascalCase for interfaces and types: `TimelineEventsTable`, `TimelineSessionsTable`, `ImageUploadResult`, `TaskState`
- Types defined inline when not shared across files
- Generic type parameters: Single uppercase letters (T, U, K, V)
- Type exports use same naming as interfaces

## Code Style

**Formatting:**
- Tool: Prettier (with oxc plugin)
- Indentation: 2 spaces
- Line width: 80 characters
- Parser: TypeScript
- Plugins: `@prettier/plugin-oxc`, `@trivago/prettier-plugin-sort-imports`

**Linting:**
- Primary: oxlint (with `--type-aware` flag)
- Fallback: ESLint
- Config: `eslint.config.js` (flat format)
- TanStack Query plugin enabled for hooks rules

**Key Settings:**
```prettier
{
  "indentStyle": "space",
  "indentWidth": 2,
  "lineWidth": 80,
  "importOrderSeparation": true,
  "importOrderSortSpecifiers": true,
  "importOrderCaseInsensitive": true
}
```

## Import Organization

**Order:**
1. Special instruments: `./instrument`
2. Third-party modules
3. @hypr/* scoped packages (internal)
4. @/* path aliases (relative to project)
5. Relative imports: `./` and `../`

**Path Aliases:**
- `@/` → Project root
- `@hypr/` → Workspace packages
- Imports use full paths, no barrel re-exports for local modules

**Separation:**
- Groups separated by blank lines
- Specifiers sorted alphabetically
- Organized by semantic grouping (React, store, components, utils)

**Example from `useKeywords.ts`:**
```typescript
import { Effect, Option, pipe } from "effect";
import type { UnknownException } from "effect/Cause";
import { toString } from "nlcst-to-string";
import { useMemo } from "react";

import * as main from "../store/tinybase/store/main";
```

## Error Handling

**Patterns:**
- Explicit error type checking: `if (error instanceof AuthSessionMissingError) { ... }`
- Error codes checked against known lists: `fatalCodes.includes(error.code ?? "")`
- Try-catch used for initialization/setup only: `try { await setupFn(); } catch { /* ignore */ }`
- Silent catch blocks documented: `} catch { // Ignore storage errors }`
- Return-based error handling in server functions: `if (error) { return { error: true, message: error.message }; }`
- No throwing errors in most functions; instead return error objects

**From `auth/errors.ts`:**
```typescript
export const isFatalSessionError = (error: unknown): boolean => {
  if (error instanceof AuthSessionMissingError) {
    return true;
  }
  if (error instanceof AuthApiError) {
    const fatalCodes = ["refresh_token_not_found", "refresh_token_already_used"];
    return fatalCodes.includes(error.code ?? "");
  }
  return false;
};

export const clearAuthStorage = async (): Promise<void> => {
  try {
    await authCommands.clear();
  } catch {
    // Ignore storage errors
  }
};
```

## Logging

**Framework:** No explicit logging library; console methods used rarely

**Patterns:**
- No console.log in production code
- Debug output via browser dev tools only
- Error context provided through return values and exception handling

## Comments

**When to Comment:**
- Avoid comments entirely as default
- Only write comments explaining "Why", never "What"
- Complex business logic that isn't self-evident
- Workarounds and non-obvious decisions

**No Comments Examples:**
- Function bodies that are clear from naming and types
- Obvious loops and iterations
- Standard React hooks usage

**Comment Examples (when needed):**
```typescript
// Ignore storage errors - may fail on restricted filesystems
```

## Function Design

**Size:**
- Small focused functions: <30 lines typical
- Complex operations broken into smaller helpers
- Example: `useKeywords` is 11 lines main, with 8 separate helper functions

**Parameters:**
- Single object parameter preferred for functions with >2 params
- Inline types for function props (avoid interfaces unless shared)
- React component props defined inline in most cases

**Return Values:**
- Tuple returns for multiple values of same type
- Object returns for multiple different values
- Promise return types explicitly annotated: `Promise<void>`, `Promise<Result>`
- Type predicates return `boolean` and use type guards: `is*` functions

**Example from `auth/errors.ts`:**
```typescript
export const isFatalSessionError = (error: unknown): boolean => { ... }
```

## Module Design

**Exports:**
- Selective named exports (not default exports)
- Functions exported directly: `export const functionName = ...`
- No barrel files for component re-exports (import directly from files)
- Index.ts used for package-level exports only

**Barrel Files Pattern (`packages/*/src/index.ts`):**
```typescript
export const { tools, toolsByName, registerTool } = registry;
export { executeCodeTool, loopsTool, posthogTool, stripeTool };
```

**Co-location:**
- Test files colocated with source: `useKeywords.ts` + `useKeywords.test.ts`
- Types defined in same file as functions
- Utilities grouped in same directory

## Class Names & Conditional Styling

**cn() utility:**
- Use `cn` from `@hypr/utils` (similar to clsx)
- Always pass arrays
- Split by logical grouping
- Example from codebase:
```typescript
className="w-full justify-start px-1 text-neutral-600 h-6"

// With cn and conditions:
const classes = cn([
  "w-full justify-start px-1 h-6",
  "text-neutral-600",
  isActive && "bg-blue-50",
]);
```

## State Management

**Patterns (from AGENTS.md):**
- Never do manual state management for forms (no `setError` pattern)
- Use `useForm` from TanStack Form for form state
- Use `useQuery`/`useMutation` from TanStack Query for async operations
- TanStack Query plugin enabled in ESLint for hooks validation

**Example from `auth.ts`:**
```typescript
export const doAuth = createServerFn({ method: "POST" })
  .inputValidator(
    shared.extend({
      provider: z.enum(["google", "github"]),
      rra: z.boolean().optional(),
    }),
  )
  .handler(async ({ data }) => {
    // Handler logic
  });
```

## Validation

**Schema Validation:**
- Zod for input validation on server functions
- Type-safe schema composition with `extend()`
- Examples: `z.enum()`, `z.string().email()`, `z.string().min(6)`
- Used in `createServerFn` input validators

## Formatting & Tooling

**dprint usage (from AGENTS.md):**
- Run `dprint fmt` after code changes
- Do NOT use `cargo fmt` (use dprint instead)
- Configured via `dprint.json` at project root
- Supports: Go, Rust, Swift, JavaScript/TypeScript, MDX, YAML, TOML, SQL, Dockerfile

**TypeScript checking (from AGENTS.md):**
- After significant changes, run `pnpm -r typecheck`
- Each package has own tsconfig.json

---

*Conventions analysis: 2026-02-11*
