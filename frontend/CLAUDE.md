# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working in the `frontend/` directory.

## Commands

```bash
npm install          # install dependencies
npm run dev          # start dev server (Vite)
npm run build        # production build
npm run lint         # ESLint
npm run typecheck    # tsc --noEmit
npm test             # run tests (Vitest)
```

## Folder Structure

```
src/
  components/    # React components, one per file
  hooks/         # custom hooks (useLesson.ts, useQuiz.ts)
  services/      # API client, external integrations
  domain/        # pure business logic, types, validation
  test/          # shared test helpers (only after duplication across 2+ test files)
```

## Naming Conventions

| What | Convention | Example |
|---|---|---|
| React components | PascalCase file and export | `QuizCard.tsx` → `export const QuizCard` |
| Hooks | camelCase with `use` prefix | `useLesson.ts` → `export const useLesson` |
| Services, utils, domain logic | camelCase | `apiClient.ts`, `calculateScore.ts` |
| Types and type files | PascalCase for types, camelCase for file | `types.ts` → `type VocabularyCard` |
| Test files | co-located, `.test.ts` suffix | `calculateScore.test.ts` next to `calculateScore.ts` |

---

## TypeScript Conventions

### Strictness

Use `strict: true` in tsconfig. Never use `any` — prefer `unknown` and narrow with type guards. Never use `@ts-ignore`; use `@ts-expect-error` with a reason comment only as a last resort.

### Types over interfaces

Use `type` for everything: data shapes, unions, intersections, mapped types, and service contracts. This keeps the codebase consistent and avoids the question of when to switch between `type` and `interface`:

```ts
// Data shape
type VocabularyCard = {
  readonly id: string;
  readonly german: string;
  readonly english: string;
  readonly difficulty: Difficulty;
};

// Union
type Difficulty = "A1" | "A2" | "B1" | "B2";

// Service contract — a type of function signatures, not a class interface
type LessonRepository = {
  readonly findById: (id: string) => Promise<Lesson | null>;
  readonly save: (lesson: Lesson) => Promise<Lesson>;
};
```

### Prefer narrowing over casting

Use discriminated unions, `in` checks, and exhaustive switch statements rather than `as` casts. Write exhaustive switches with a `never` default to catch unhandled variants at compile time. Apply this consistently — including in reducers:

```ts
function getDifficultyLabel(d: Difficulty): string {
  switch (d) {
    case "A1": return "Beginner";
    case "A2": return "Elementary";
    case "B1": return "Intermediate";
    case "B2": return "Upper Intermediate";
    default: {
      const _exhaustive: never = d;
      throw new Error(`Unhandled difficulty: ${_exhaustive}`);
    }
  }
}
```

### Readonly by default

Mark properties and arrays as `readonly` on domain types and state types. Skip `readonly` on React props — props are already structurally immutable and marking every callback `readonly` adds noise without catching real bugs:

```ts
// Domain type — readonly matters, prevents accidental mutation
type QuizState = {
  readonly currentIndex: number;
  readonly answers: readonly Answer[];
  readonly score: number;
};

// React props — no readonly needed
type QuizCardProps = {
  card: VocabularyCard;
  onAnswer: (correct: boolean) => void;
  disabled?: boolean;
};
```

---

## Error Handling

### Services and domain logic

Use explicit return types for operations that can fail. Throw typed errors at service boundaries; let unexpected errors propagate naturally:

```ts
class AppError extends Error {
  constructor(
    message: string,
    readonly code: "NOT_FOUND" | "VALIDATION" | "NETWORK" | "UNAUTHORIZED",
  ) {
    super(message);
    this.name = "AppError";
  }
}

const fetchLesson = async (id: string): Promise<Lesson> => {
  const response = await apiClient.get(`/api/lessons/${id}`);
  if (!response.ok) {
    throw new AppError(`Lesson ${id} not found`, "NOT_FOUND");
  }
  return response.json();
};
```

### React error boundaries

Wrap top-level routes in an error boundary. Hooks that call services should catch `AppError` and expose error state — components never try/catch directly:

```ts
const useLesson = (id: string): UseLessonReturn => {
  const [state, setState] = useState<UseLessonReturn>({
    lesson: null,
    loading: true,
    error: null,
  });

  useEffect(() => {
    fetchLesson(id)
      .then((lesson) => setState({ lesson, loading: false, error: null }))
      .catch((e) =>
        setState({
          lesson: null,
          loading: false,
          error: e instanceof AppError ? e.message : "Something went wrong",
        }),
      );
  }, [id]);

  return state;
};
```

---

## SOLID and Functional Principles

These principles reinforce each other in TypeScript. The goal is small, composable, testable units.

### Single Responsibility

Each module, function, and component does one thing. If a function needs an "and" to describe it, split it. Prefer many small files over few large ones.

### Open/Closed via composition

Extend behavior by composing functions and types, not by modifying existing code. Use union types to model variants and higher-order functions to extend behavior:

```ts
// Open for extension via union expansion
type CardEvent =
  | { type: "answered"; cardId: string; correct: boolean }
  | { type: "skipped"; cardId: string }
  | { type: "flagged"; cardId: string; reason: string };

// Open for extension via composition
const withRetry = <T>(
  fn: () => Promise<T>,
  attempts = 3,
): (() => Promise<T>) => {
  if (attempts < 1) throw new Error("attempts must be >= 1");
  return async () => {
    for (let i = 0; i < attempts; i++) {
      try { return await fn(); }
      catch (e) { if (i === attempts - 1) throw e; }
    }
    throw new Error("Unreachable");
  };
};
```

### Liskov Substitution

When using discriminated unions, every variant must satisfy the contract fully. Don't add a variant that silently ignores part of the expected behavior. If a function accepts `CardEvent`, every event type must be meaningful in that context.

### Interface Segregation

Keep type signatures small. A function that only needs a card's ID and text should accept `{ id: string; german: string }` as an inline type or a dedicated narrow type — not the full `VocabularyCard`. Use `Pick<>` sparingly; prefer a named narrow type when the subset has a clear meaning, inline when it's a one-off:

```ts
// Named narrow type — this concept is reused
type CardPreview = {
  readonly id: string;
  readonly german: string;
};

// Inline — one-off function signature
const logCard = (card: { id: string; german: string }) => { /* ... */ };
```

### Dependency Inversion

Depend on abstractions (types, function signatures), not concretions. Pass dependencies as function parameters or React context rather than importing singletons directly. This is the single biggest lever for testability:

```ts
type FetchCards = (lessonId: string) => Promise<VocabularyCard[]>;

const buildQuiz = async (
  lessonId: string,
  fetchCards: FetchCards,
): Promise<Quiz> => {
  const cards = await fetchCards(lessonId);
  return { cards, currentIndex: 0, score: 0 };
};
```

### Functional defaults

Prefer pure functions: same input, same output, no side effects. Push side effects to the edges (API calls, event handlers, top-level orchestration). Use `map`, `filter`, `reduce` over mutation loops. Return new objects instead of mutating — the `readonly` convention enforces this naturally.

Avoid classes for business logic. Use plain functions and closures. The only class in the codebase should be `AppError` (above) and anything a framework requires.

---

## React Component Conventions

### Component structure

Use function components exclusively. Never use `React.FC` — use plain function signatures with typed props. One exported component per file. Small internal sub-components (a list item renderer, a styled wrapper) may live in the same file if they're only used by the parent.

Name the file after the component: `QuizCard.tsx` exports `QuizCard`.

Organize component files in this order:
1. Imports
2. Types (props, local types)
3. Helper functions (defined before the component so the reader sees definitions before usage)
4. Component function (exported)

### Props

Define props as a `type`. Destructure in the function signature. Avoid spreading arbitrary props — be explicit about what the component accepts:

```tsx
type QuizCardProps = {
  card: VocabularyCard;
  onAnswer: (correct: boolean) => void;
  disabled?: boolean;
};

export const QuizCard = ({ card, onAnswer, disabled = false }: QuizCardProps) => {
  // ...
};
```

### State management

Keep state as local as possible. Lift state only when two sibling components need it. Use `useReducer` over `useState` when state transitions are complex or interdependent — it makes the state machine explicit and testable outside React:

```ts
type QuizAction =
  | { type: "answer"; correct: boolean }
  | { type: "next" }
  | { type: "reset" };

const quizReducer = (state: QuizState, action: QuizAction): QuizState => {
  switch (action.type) {
    case "answer":
      return { ...state, score: state.score + (action.correct ? 1 : 0) };
    case "next":
      return { ...state, currentIndex: state.currentIndex + 1 };
    case "reset":
      return { currentIndex: 0, answers: [], score: 0 };
    default: {
      const _exhaustive: never = action;
      throw new Error(`Unhandled action: ${JSON.stringify(_exhaustive)}`);
    }
  }
};
```

### Custom hooks

Extract reusable logic into custom hooks. A hook should do one thing. Name it `use<Thing>`. Keep the return type explicit:

```ts
type UseLessonReturn = {
  lesson: Lesson | null;
  loading: boolean;
  error: string | null;
};

const useLesson = (lessonId: string): UseLessonReturn => { /* ... */ };
```

### Conditional rendering

Prefer early returns for loading/error states over deeply nested ternaries. This keeps the happy-path JSX flat and readable:

```tsx
export const LessonView = ({ lessonId }: LessonViewProps) => {
  const { lesson, loading, error } = useLesson(lessonId);

  if (loading) return <Spinner />;
  if (error) return <ErrorBanner message={error} />;
  if (!lesson) return null;

  return (
    <div>
      <h1>{lesson.title}</h1>
      <CardList cards={lesson.cards} />
    </div>
  );
};
```

### Component size

If a component exceeds ~80 lines of JSX or manages more than three pieces of independent state, break it apart. Extract visual sub-sections into child components and complex logic into hooks.

### Avoid

- `useEffect` for derived state — compute it inline or with `useMemo`
- Prop drilling beyond two levels — use context or composition (children, render props)
- `any` in event handlers — type them (`React.ChangeEvent<HTMLInputElement>`)
- Index as key in lists where items can reorder
- Business logic inside components — extract to pure functions in `domain/` and test independently

---

## Testing

Tests use Vitest and live next to the source file (`calculateScore.test.ts` beside `calculateScore.ts`). Shared test helpers go in `src/test/`. See the `vitest-unit-tests` skill for full conventions on structure, mocking, assertions, and async patterns.
