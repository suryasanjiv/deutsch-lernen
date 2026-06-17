---
name: vitest-unit-tests
description: Create, update, fix, refactor, or improve Vitest unit tests for TypeScript, JavaScript, React, Node, service, utility, and library code.
when_to_use: >
  Use when the user mentions Vitest, vi, describe/it/expect, unit tests, test coverage,
  failing tests, regression tests, .test.ts, .spec.ts, .test.tsx, or .spec.tsx.
  Also use when production code changes and matching Vitest tests likely need to be added
  or updated. In greenfield projects, establish a clear, consistent test style instead of
  relying on pre-existing conventions. Do not use for Playwright/Cypress end-to-end tests
  or Jest-only projects unless migrating to Vitest.
argument-hint: "[target file, behavior to cover, failing test output, or feature]"
allowed-tools:
  - Read
  - Grep
  - Glob
  - Edit
  - Write
  - Bash
---

# Vitest Unit Tests

Create, update, repair, or improve Vitest unit tests.

If the project already has tests, match their established style. If it's greenfield, establish a clean, consistent one and preserve it.

Target request: `$ARGUMENTS`

---

## Conventions

### File placement

Place `*.test.ts` or `*.test.tsx` next to the file under test. Shared helpers go in `src/test/` only after duplication appears across two or three test files.

### Imports and structure

Always use explicit Vitest imports:

```ts
import { describe, it, expect, vi, beforeEach, afterEach } from "vitest";
```

Use `describe` for the unit under test. Write `it` names that state the outcome and condition:

```ts
it("returns eligible when all required checks pass", () => {});
it("throws when the request is missing an account id", () => {});
```

Vague names like `"works"` or `"handles input"` make failures unreadable — avoid them.

### Arrange / Act / Assert

Use AAA comments when the test has meaningful setup. Skip them for one-liner assertions where the flow is obvious.

---

## What to test

Focus on **observable behavior**: return values, thrown errors, rejected promises, validation messages, rendered output, emitted events.

For each new function or module, cover at minimum:
1. Happy path
2. One meaningful edge case
3. One failure or invalid-input case (when applicable)

Avoid testing private internals — if a test only proves a mock was called without checking the resulting behavior, it's coupling to implementation.

---

## Assertions

Prefer the most specific matcher available. `toEqual` for objects/arrays, `toBe` for primitives, `toThrow` / `rejects.toThrow` for errors. Reserve `toBeDefined` and `toBeTruthy` for cases where existence or truthiness *is* the behavior being asserted.

---

## Async

Always `await` async behavior. Use `await expect(...).rejects.toThrow(...)` for rejected promises. Never use real sleeps or arbitrary timeouts. Use fake timers only when time-dependent behavior is being tested, and always restore them:

```ts
afterEach(() => { vi.useRealTimers(); });
```

---

## Mocking

Mock **boundaries** (network, database, filesystem, external SDKs, injected services, timers, date/random/UUID). Do not mock the unit under test or simple pure helpers.

Use `vi.fn()` for injected dependencies, `vi.spyOn()` for observing existing objects, `vi.mock()` only when DI is unavailable.

Restore mocks between tests:

```ts
afterEach(() => { vi.restoreAllMocks(); });
```

---

## Test data

Keep data inline and minimal. Extract a factory function only when repeated setup becomes noisy across multiple tests:

```ts
const createRequest = (overrides = {}) => ({
  accountId: "account-123",
  instrumentId: "instrument-456",
  quantity: 10,
  ...overrides,
});
```

Don't hide important test differences inside complex builders — the reader should see what matters in each test at a glance.

---

## React components

When testing React components, use React Testing Library and prefer user-visible queries (`getByRole`, `getByText`) over implementation details like class names. Test what the user sees and does: rendered text, button clicks, form behavior, validation, loading states, error states. Mock API boundaries, not the component.

---

## Updating existing tests

Preserve existing coverage unless clearly obsolete. Add new cases near related ones. Never delete a failing test just to make the suite pass, and never weaken assertions unless the old assertion was wrong. Keep unrelated formatting churn out of the diff.

If a new test reveals a **production bug**: keep the test as a regression case, explain the bug briefly, make the smallest fix needed (if implementation was requested), and re-run.

---

## Verification

Detect the package manager from lockfiles (`pnpm-lock.yaml` → pnpm, `yarn.lock` → yarn, `package-lock.json` → npm, `bun.lock*` → bun) and run the narrowest test command:

```bash
pnpm vitest run path/to/file.test.ts
```

If no test script exists, add one:

```json
{ "scripts": { "test": "vitest run", "test:watch": "vitest" } }
```

On failure: read the output carefully, fix test-side issues, and never skip or loosen assertions to hide real failures. Report unrelated blockers clearly.

---

## Completion checklist

Before finishing, confirm:

- File naming and placement are consistent
- Tests cover observable behavior (happy path, edge case, failure)
- Assertions are specific
- Async is properly awaited
- Mocks are scoped and restored
- Test data is readable and minimal
- Verification command was run (or the reason it couldn't is stated)

Respond with: tests created/updated, scenarios covered, verification command and result, and any concerns (uncovered behavior, flaky setup, production bugs found).