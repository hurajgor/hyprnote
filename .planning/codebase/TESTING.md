# Testing Patterns

**Analysis Date:** 2026-02-11

## Test Framework

**Runner:**
- Vitest 3.2.4+
- Config files: `vitest.config.ts` per workspace
- Entry: `test/**/*.test.ts` pattern in each package

**Configuration Example (`apps/bot/vitest.config.ts`):**
```typescript
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    include: ["test/**/*.test.ts"],
    setupFiles: ["test/setup-env.ts"],
    passWithNoTests: true,
    coverage: {
      provider: "v8",
    },
  },
});
```

**Assertion Library:**
- Vitest built-in `expect()` API
- No additional assertion libraries

**Run Commands:**
```bash
pnpm test                    # Run all tests in workspace
vitest run                   # Run tests in single package (no watch)
vitest                       # Watch mode
vitest run --coverage        # Generate coverage report
npm run test -w package-name # Run specific workspace package
```

## Test File Organization

**Location:**
- Colocated with source files (same directory)
- Same filename with `.test.ts` suffix

**Naming Pattern:**
```
src/
  ├── hooks/
  │   ├── useKeywords.ts
  │   └── useKeywords.test.ts
  ├── utils/
  │   ├── timeline.ts
  │   └── timeline.test.ts
  └── store/
      └── persister/
          ├── persister.ts
          └── persister.test.ts
```

**File Types:**
- `.test.ts` for unit/integration tests (standard)
- `.spec.ts` for E2E tests (in e2e/blackbox/tests directory)

## Test Structure

**Suite Organization - Pattern from `useKeywords.test.ts`:**
```typescript
import { describe, expect, it } from "vitest";

import { extractKeywordsFromMarkdown } from "./useKeywords";

describe("extractKeywordsFromMarkdown", () => {
  const cases: Array<{
    description: string;
    input: string;
    keywords: string[];
    keyphrases: string[];
  }> = [
    {
      description: "extracts hashtags from markdown",
      input: "This is #awesome and #cool stuff",
      keywords: ["awesome", "cool"],
      keyphrases: [],
    },
    // More test cases...
  ];

  it.each(cases)("$description", ({ input, keywords, keyphrases }) => {
    const result = extractKeywordsFromMarkdown(input);
    expect(result.keywords).toEqual(expect.arrayContaining(keywords));
    expect(result.keyphrases).toEqual(expect.arrayContaining(keyphrases));
  });
});
```

**Describe/Test/It Pattern:**
- `describe()` for test suites
- `it()` or `test()` for individual tests
- Nested `describe()` for logical grouping
- Example from `timeline.test.ts`:
```typescript
describe("timeline utils", () => {
  test("getBucketInfo returns Today for current date", () => {
    const info = getBucketInfo(new Date("2024-01-15T05:00:00.000Z"));
    expect(info).toMatchObject({ label: "Today", precision: "time" });
  });
});
```

**Setup/Teardown Pattern (`timeline.test.ts`):**
```typescript
describe("timeline utils", () => {
  beforeEach(() => {
    vi.useFakeTimers();
    vi.setSystemTime(SYSTEM_TIME);
  });

  afterEach(() => {
    vi.useRealTimers();
  });

  test("...", () => { ... });
});
```

## Mocking

**Framework:**
- Vitest `vi` module for mocking
- `vi.hoisted()` for module-level mocks
- `vi.mock()` for importing mocked modules

**Mocking Pattern (`persister.test.ts`):**
```typescript
const settingsMocks = vi.hoisted(() => ({
  vaultBase: vi
    .fn()
    .mockResolvedValue({ status: "ok", data: "/mock/data/dir/hyprnote" }),
}));

const fsSyncMocks = vi.hoisted(() => ({
  deserialize: vi.fn(),
  serialize: vi.fn().mockResolvedValue({ status: "ok", data: "" }),
  writeDocumentBatch: vi.fn().mockResolvedValue({ status: "ok", data: null }),
  readDocumentBatch: vi.fn(),
  cleanupOrphan: vi.fn().mockResolvedValue({ status: "ok", data: 0 }),
}));

vi.mock("@hypr/plugin-settings", () => ({ commands: settingsMocks }));
vi.mock("@hypr/plugin-fs-sync", () => ({ commands: fsSyncMocks }));
vi.mock("@hypr/plugin-fs2", () => ({ commands: fs2Mocks }));
```

**Mock Clearing:**
```typescript
beforeEach(() => {
  store = createTestMainStore();
  vi.clearAllMocks();
});
```

**What to Mock:**
- External service calls (API clients, storage)
- Module imports (other services, plugins)
- Async operations
- System time and timers

**What NOT to Mock:**
- Core language features
- Pure utility functions
- Standard libraries
- The function under test

## Fixtures and Factories

**Test Data Creation:**
- Helper functions for creating test data
- Example: `createTestMainStore()` from `testing/mocks`
- Example: Mock data directories: `MOCK_DATA_DIR`

**Fixtures Location:**
- `testing/mocks` directories in test structure
- Colocated exports from test files

**Pattern from `as-tables-changes.test.ts`:**
```typescript
describe("asTablesChanges", () => {
  describe("e2e: applying changes to store", () => {
    let store: ReturnType<typeof createMergeableStore>;

    beforeEach(() => {
      store = createMergeableStore("test-store");
    });

    test("wraps tables for merging", async () => {
      store.setCell("users", "user-1", "name", "Alice");
      // Test assertions...
    });
  });
});
```

## Coverage

**Requirements:**
- V8 provider configured
- `passWithNoTests: true` allows zero-test packages

**View Coverage:**
```bash
vitest run --coverage
```

## Test Types

**Unit Tests:**
- Isolated function testing
- Location: Colocated with source
- Scope: Single function or module
- Example: `useKeywords.test.ts` tests `extractKeywordsFromMarkdown` function

**Integration Tests:**
- Multiple components working together
- Location: Colocated with source
- Scope: Module interactions
- Example: `persister.test.ts` tests persister with mocked plugins
- Example: `timeline.test.ts` tests bucketBuilding with multiple helper functions
- Pattern: Named as "e2e: ..." in describe blocks:
```typescript
describe("e2e: applying changes to store", () => {
  // Integration test cases
});
```

**E2E Tests:**
- Framework: Playwright
- Location: `e2e/blackbox/tests/` directory
- Pattern: `.spec.ts` files
- Example: `e2e/blackbox/tests/app.spec.ts`
- Run: Via `--play` flag or separate test runner

## Common Patterns

**Async Testing:**
```typescript
test("configures correct table and directory names", async () => {
  fsSyncMocks.readDocumentBatch.mockResolvedValue({
    status: "ok",
    data: {},
  });

  const persister = createOrganizationPersister(store);
  await persister.load();

  expect(fsSyncMocks.readDocumentBatch).toHaveBeenCalledWith(
    `${MOCK_DATA_DIR}/organizations`,
  );
});
```

**Parametrized Tests with `it.each()`:**
```typescript
const cases: Array<{
  description: string;
  input: string;
  expected: any[];
}> = [
  { description: "case 1", input: "...", expected: [...] },
  { description: "case 2", input: "...", expected: [...] },
];

it.each(cases)("$description", ({ input, expected }) => {
  const result = process(input);
  expect(result).toEqual(expect.arrayContaining(expected));
});
```

**Error Testing:**
```typescript
test("returns null for undefined", () => {
  expect(extractChangedTables(undefined)).toBeNull();
});

test("returns null for empty array", () => {
  expect(extractChangedTables([] as any)).toBeNull();
});

test("returns null for non-array input", () => {
  expect(extractChangedTables("string" as any)).toBeNull();
  expect(extractChangedTables(123 as any)).toBeNull();
});
```

**Fake Timers Pattern:**
```typescript
const SYSTEM_TIME = new Date("2024-01-15T12:00:00.000Z");

describe("timeline utils", () => {
  beforeEach(() => {
    vi.useFakeTimers();
    vi.setSystemTime(SYSTEM_TIME);
  });

  afterEach(() => {
    vi.useRealTimers();
  });

  test("getBucketInfo returns Today for current date", () => {
    const info = getBucketInfo(new Date("2024-01-15T05:00:00.000Z"));
    expect(info).toMatchObject({ label: "Today", precision: "time" });
  });
});
```

**Store Testing Pattern:**
```typescript
test("iterates over rows from store content", () => {
  store.transaction(() => {
    store.setCell("sessions", "session-1", "title", "Meeting Notes");
    store.setCell("sessions", "session-1", "created_at", "2024-01-01");
  });

  const tables = store.getTables() as GenericTablesContent;
  const rows = iterateTableRows(tables, "sessions");

  expect(rows).toHaveLength(2);
  expect(rows.find((r) => r.id === "session-1")).toMatchObject({
    id: "session-1",
    title: "Meeting Notes",
    created_at: "2024-01-01",
  });
});
```

**Defensive Input Testing Pattern:**
```typescript
describe("defensive input handling", () => {
  test("returns null for undefined", () => {
    expect(extractChangedTables(undefined)).toBeNull();
  });

  test("returns null for null", () => {
    expect(extractChangedTables(null as any)).toBeNull();
  });

  test("returns null for empty array", () => {
    expect(extractChangedTables([] as any)).toBeNull();
  });

  test("returns null for non-array input", () => {
    expect(extractChangedTables("string" as any)).toBeNull();
    expect(extractChangedTables(123 as any)).toBeNull();
  });
});
```

## React Component Testing

**Testing Library Pattern (`tinybase.test.tsx`):**
```typescript
import { renderHook } from "@testing-library/react";
import { describe, expect, it } from "vitest";

describe("useHookName", () => {
  it("returns expected value", () => {
    const { result } = renderHook(() => useHookName());
    expect(result.current).toBeDefined();
  });
});
```

**Hook Testing:**
- Use `renderHook` from `@testing-library/react`
- Test hook return values
- Test side effects and state updates
- Location: Colocated with hook file

## Testing Best Practices

**Descriptive Test Names:**
- Test descriptions should explain the behavior, not the implementation
- Use full sentences in describe/it blocks
- Examples:
  - ✅ "extracts hashtags from markdown"
  - ✅ "handles empty text gracefully"
  - ❌ "test function"
  - ❌ "works"

**Arrange-Act-Assert:**
- Setup data (Arrange)
- Call the function (Act)
- Check the result (Assert)

**One Assertion Focus:**
- Tests can have multiple assertions
- But focused on testing one behavior
- Use `describe()` to group related tests

**Test Isolation:**
- Each test independent
- Use `beforeEach()` for setup
- Use `afterEach()` for cleanup
- Clear mocks between tests

---

*Testing analysis: 2026-02-11*
