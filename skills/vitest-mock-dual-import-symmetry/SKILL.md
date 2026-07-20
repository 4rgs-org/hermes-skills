---
name: vitest-mock-dual-import-symmetry
description: vitest `vi.mock` factories must provide BOTH `default` and named exports for any module that some files import as `import x from "…"` and other files import as `import { x } from "…"`. Use when a vi.mock test fails with `No "default" export is defined on the "<module>" mock` or `undefined is not a function` after a previously-passing test starts rendering a new component that uses the same hook/utility in a different import style. Covers why the mock-factory is not bidirectional by default, the worked example from RenderCard.playAgainOverlay.test.tsx (June 2026), the dual-import fix pattern, the "single-ownership of cross-session flags" extension that prevents this class from recurring, and the vitest 4 `vi.fn<T>(impl)` single-type-arg signature change that broke legacy two-arg type annotations.
version: 1.1.0
author: Hermes Agent
license: MIT
platforms: [macos, linux]
metadata:
  hermes:
    tags: [vitest, react, testing, mocking, app_web, odizey]
    related_skills: [debugging-odizey-useeventform-validation-tests, odizey-react-hooks-stability]
---

# vitest — Dual-Import Symmetry in `vi.mock` Factories

## The trap

A vitest `vi.mock("@/module", () => ({ … }))` factory replaces the
**module record**, not individual bindings. If a module is
imported in two different styles across the dependency tree:

```ts
// file A
import useSecureEventNavigation from "@/hooks/useSecureEventNavigation";

// file B
import { useSecureEventNavigation } from "@/hooks/useSecureEventNavigation";
```

…then a `vi.mock` that only provides one of those bindings (e.g.
only `default: () => ({…})`) will satisfy one of the import sites
and crash on the other:

```
Error: No "default" export is defined on the "@/hooks/useSecureEventNavigation" mock.
Did you forget to return it from "vi.mock"?
```

or its symmetric cousin:

```
TypeError: undefined is not a function
    at RenderCardBase (RenderCard.tsx:207:5)
    // … destructure of useSecureEventNavigation() returned undefined
```

Both errors say the same thing: the test's mock factory is
half-complete relative to the import styles the SUT actually uses.

## Why this fires specifically in cross-file mocks

ESLint's `import/no-default-export` and TypeScript's
`esModuleInterop` make it perfectly legal to mix default and named
imports of the same module from different files. The module
author may have `export const foo` AND `export default foo`, and
callers may consume either form. The mock factory has to provide
**both** bindings or one of the consumers will fail.

In a tightly-coupled component tree (a `useSecureEventNavigation`
hook used by a page and a modal that the page renders), the
import styles drift apart organically:

- The page imports the hook as `import useSecureEventNavigation
  from "..."` (default, ergonomic for a single export).
- The modal imports it as `import { useSecureEventNavigation }
  from "..."` (named, ergonomic alongside other named imports).

The first file to write the test usually has the default form
mocked, and the second file is where the test starts to fail.

## The fix pattern (apply in the same iteration)

```ts
vi.mock("@/hooks/useSecureEventNavigation", () => ({
  // Both `import x from "..."` (default) and
  // `import { useSecureEventNavigation } from "..."` (named) are
  // used in the dependency tree (e.g. `RenderCard.tsx` does the
  // default import, `PlayUIModeSelector.tsx` does the named
  // import), so the mock has to provide both. Returning the same
  // hook function for both is safe — the call sites destructure
  // the returned object, not the import binding.
  default: () => ({
    navigateToEventDetail: vi.fn(),
    navigateToGame: vi.fn(),
    navigateToShop: vi.fn(),
  }),
  useSecureEventNavigation: () => ({
    navigateToEventDetail: vi.fn(),
    navigateToGame: vi.fn(),
    navigateToShop: vi.fn(),
  }),
}));
```

Both keys point to the **same function** (or factories that build
the same shape). That matters because:

- The hook may be called twice (once via default import, once via
  named import) and both calls must return the same shape so the
  destructuring at the call site succeeds.
- `vi.fn()` instances are stable across mock re-imports within a
  test, so the test can `expect(thing.navigateToGame).toHaveBeenCalledWith(...)`
  regardless of which import style reached the mock.

## Worked example (June 2026, `app_web`)

The bug surfaced in
`src/components/CardEvents/components/RenderCard.playAgainOverlay.test.tsx`.
The SUT was `RenderCard.tsx`. When the test was first written,
`RenderCard` did not render `<PlayUIModeSelector>` in this
overlay flow, so the test passed with a single `default:` export
on the mock.

Later, a separate change added `<PlayUIModeSelector open={…} />`
to `RenderCard.tsx`'s JSX. The modal mounts at all times
(controlled by `open`), so even a `pre_question` overlay test
that doesn't open the modal still triggers the modal's
`useEffect` body, which calls `useSecureEventNavigation()` via the
**named** import. The mock only had `default:`, so the modal's
named import returned `undefined`, the destructure crashed, and
the test reported `Cannot access 'useSecureEventNavigation'
before initialization` (or `TypeError: undefined is not a
function`).

The fix was a 2-line addition to the mock factory — see the
template above. After the fix, both the page's default import
and the modal's named import receive the same mock function.

## Detection recipe (under 1 minute)

When a test that has been passing for weeks starts failing with
one of these:

- `No "<X>" export is defined on the "<module>" mock. Did you
  forget to return it from "vi.mock"?`
- `Cannot access '<X>' before initialization` inside a component
  that was not edited in the current diff.
- `TypeError: undefined is not a function` on a hook call in a
  component that was not edited in the current diff.

…do the following:

1. Open the failing test file. Find the `vi.mock("@/module", …)`
   for the module whose named/default export is "missing".
2. Open the dependency graph. Search the component tree mounted
   by the test (transitively, including all sub-components) for
   `from "@/module"` — both default-style and named-style imports.
3. The mock factory must provide keys for every import style found.
4. If multiple styles are present, point them to the same
   function (or factory).
5. Re-run focused, then full.

## Anti-patterns to avoid

- **Don't** add only the export style that's mentioned in the
  error. The error names the missing style, but the test may also
  be using the other style silently — adding only the named
  export can leave the default-import site broken and pass with
  a different downstream symptom. Always grep for both styles.
- **Don't** try to "normalize" the import style across the
  codebase. Both `import x from "…"` and `import { x } from "…"`
  are valid; the codebase has both for ergonomic reasons; do not
  force a refactor as part of fixing the mock.
- **Don't** use `vi.importActual` + spread to keep the real
  implementation. That defeats the purpose of mocking (the test
  would call the real hook, which may need to set up Redux state,
  configure routing, etc.). The fix is to provide a tiny stub
  that returns the shape the SUT expects.
- **Don't** mock the import by aliasing the default import in the
  test file (`import useSecureEventNavigation from "..."` and
  reassign it). vitest's `vi.mock` operates at the module
  boundary, not at the local import site. The factory is the
  only correct place to provide the dual export.

## Sibling pitfall 1 — i18next mocks must accept BOTH `t()` signatures

i18next supports two `t()` signatures in the wild:

```ts
t(key, defaultValue)            // second arg is the fallback string
t(key, { defaultValue })        // second arg is an options object
```

A real `react-i18next` accepts both and treats the second arg
accordingly. A vitest mock that hardcodes one signature will
silently break when the SUT uses the other. Symptom: rendered
text equals the fallback string verbatim (e.g. tabs render
`"points"/"trivias"/`...` instead of `"Puntos"/"Trivias completas"/...`)
because the mock returns the key in the unhandled branch. The test
then fails with `Unable to find an accessible element with the
role "tab" and name /Puntos/i`.

Fix pattern for the mock:

```ts
vi.mock("react-i18next", () => ({
  useTranslation: () => ({
    t: (key: string, fallback?: string | { defaultValue?: string }): string => {
      if (typeof fallback === "string") return fallback;
      if (fallback && "defaultValue" in fallback) return fallback.defaultValue ?? key;
      return key;
    },
    i18n: { language: "es" },
  }),
}));
```

When the SUT renders user-facing strings the assertions verify by
name (a tabs row that searches `screen.getByRole("tab", { name:
/Puntos/i })`), the mock must additionally return the real
translated string for the keys the test names — not just the key.
Build a small `Record<string, string>` in the factory mirroring the
values in `public/locales/<lang>/translation.json` for exactly the
keys the assertions reference. Do NOT load the real JSON via
`vi.importActual` (slow, brittle, requires the full i18n backend
to be initialized). Mirror just the keys the test needs.

Worked example (July 2026, `app_web` —
`src/components/GlobalLeaderboard/__tests__/GlobalLeaderboardModal.test.tsx`).
The SUT calls `t("global-leaderboard-tabs-points", "points")`
(string fallback), the mock only handled `{ defaultValue }`, and
tabs rendered as `points`/`trivias`/`certificates`/`medals`. After
applying the dual-signature fix AND a 4-entry dict of the literal
Spanish labels (lifted from `public/locales/es/translation.json`),
both smoke tests in the file went green. Fix surface: ~50 lines
in one test file, no source changes.

## Sibling pitfall 2 — `setupFiles` drift: jest-dom matchers may not be loaded globally

When `vitest.config.ts` has `setupFiles: ["./<path>/setupTests.ts"]`
pointing at a file that exists but only polyfills `matchMedia`
(common in mid-sized Vitest migrations), matchers like
`toBeInTheDocument`, `toHaveTextContent`, `toBeVisible` are NOT
registered. Symptom:

```
Error: Invalid Chai property: toBeInTheDocument
```

…and only one half of a `describe` block fails (the half that uses
jest-dom matchers; the half that uses pure vitest matchers like
`toHaveBeenCalled` passes).

Diagnosis recipe (under 30 seconds):

1. Open `vitest.config.ts`. Read the `setupFiles` array.
2. Open each file it points to. Check if it imports
   `@testing-library/jest-dom` or `@testing-library/jest-dom/vitest`.
3. If not: the setup file is missing the jest-dom registration,
   regardless of whether `@testing-library/jest-dom` is in
   `package.json` `devDependencies`.

Fix — pick one (preferred order):

- **Local import in the test file** — minimum-blast-radius. Works
  because `import "@testing-library/jest-dom/vitest"` is a
  side-effecting import that registers the matchers globally for
  the test run. Add a one-line comment explaining WHY this isn't
  in the global setup file (so a future reviewer doesn't move it).

  ```ts
  // Local jest-dom import — the global `setupFiles` in
  // vitest.config.ts points to `./__tests__/setupTests.ts` which
  // only polyfills matchMedia, not jest-dom matchers like
  // `toBeInTheDocument`.
  import "@testing-library/jest-dom/vitest";
  ```

- **Fix the global setup file** — if the project clearly intends
  to use jest-dom across all tests, append
  `import "@testing-library/jest-dom/vitest";` to the setup file
  the config points to AND/OR add the missing one to the
  `setupFiles` array. Re-run the full suite to catch surprises.

Anti-patterns:

- Do NOT add a second `setupTests.ts` shadow file without
  deleting the broken path in `vitest.config.ts`. Two setup files
  double-register matchers and create order-dependence that
  breaks the moment the `setupFiles` array is reshuffled.
- "Fix the global setup file" is a **cross-cutting change** that
  affects every test in the repo. In multi-team repos, flag it
  in the PR description and run the full suite, not just the
  modified test, before merging — matchers registering twice can
  cause hard-to-debug "matcher already initialized" warnings.

Worked example (July 2026, `app_web` — same
`GlobalLeaderboardModal.test.tsx`). First test passed (it used
`toHaveBeenCalled`), second test failed with
`Invalid Chai property: toBeInTheDocument`. `vitest.config.ts`
pointed `setupFiles` at `./__tests__/setupTests.ts` which only
polyfilled `matchMedia`. Adding the local
`import "@testing-library/jest-dom/vitest";` brought both tests
green without touching the global config or affecting the other
361 tests in the suite.

## When the bug class extends: single-ownership of cross-session flags

A related but distinct class of bug that surfaces in the same
test files: a flag stored in `localStorage` / `secureStorage` is
read from **two different components** in the same flow, with
the second component's read happening after the first has already
mutated the flag. The first component's mock in vitest does not
block the second component from doing the same mutation in
production. Symptoms:

- "the page redirects to the wrong URL on re-mount"
- "the user lands on `/dashboard` instead of `/dashboard/events/detail/<token>`"

Fix: pick ONE component to own the consume. All other components
in the flow should be "transparent" — they don't read the flag.
The dual-import symmetry pattern doesn't fix this class, but the
same diagnostic discipline applies: read the actual call chain,
not the test's expected one. Captured in
`odizey-react-hooks-stability` under "NEVER have two components
both consume the same cross-session flag".

## When the SUT mounts a component that uses `useQuery` but the test predates React Query

Different bug class from dual-import, but the same `vi.mock` family
is the fix. When you add a new component to a page that an existing
test already renders, the new component may internally call
`useQuery` (or `useAppQuery`, the same hook in `app_web`). The
test then fails with:

```
Error: No QueryClient set, use QueryClientProvider to set one
```

This happens in two flavors:

1. The new component is mounted unconditionally at the top of the
   page (e.g. a `<StreakModalTrigger />` in `DashboardPage`). The
   test renders `DashboardPage` and now the trigger runs, calling
   `useStreakSchema()` which calls `useQuery()` without a
   `QueryClientProvider` in the test's render tree.
2. The new component is mounted conditionally but the test's
   scenario triggers the condition (e.g. `<EventCountdown />`
   always renders, even when `start_time` is null).

The two fixes, in order of preference:

### Option A: mock the entire hooks barrel in the test

When the new component uses 3+ hooks from a barrel (e.g. `@/hooks/useStreak`
exports `useStreakSchema`, `useStreakState`, `useStreakMissions`,
`useStreakClaimDaily`, `useStreakClaimMission`, `useStreakRanking`,
`useStreakTouchLastSeen`), mock the barrel with stub return values:

```ts
vi.mock("@/hooks/useStreak", () => ({
  useStreakSchema: () => ({ data: null, isLoading: false }),
  useStreakState: () => ({ data: null, isLoading: false }),
  useStreakMissions: () => ({ data: null, isLoading: false, isFetching: false }),
  useStreakClaimDaily: () => ({ mutate: vi.fn(), isPending: false }),
  useStreakClaimMission: () => ({ mutate: vi.fn(), isPending: false }),
  useStreakRanking: () => ({ data: { rows: [], self_position: null, fetched_at: "" } }),
  useStreakTouchLastSeen: () => ({ mutate: vi.fn() }),
}));
```

This works because the test doesn't care about streak data — it
only cares about the navigation/routing flow under test. The mock
satisfies the React Query requirement (no error thrown on render)
without standing up a full QueryClient.

### Option B: wrap with QueryClientProvider in `renderWithProviders`

When the test DOES care about the data (e.g. an integration test
that asserts on rendered values), wrap the test render in a
`QueryClientProvider` and create a fresh `QueryClient` per test
to keep cache state isolated:

```ts
const queryClient = new QueryClient({
  defaultOptions: {
    queries: { retry: false, gcTime: 0 },
  },
});
render(
  <QueryClientProvider client={queryClient}>
    <DashboardPage />
  </QueryClientProvider>,
);
```

**Recipe location**: if your test helpers export a
`renderWithProviders` function, add the QueryClientProvider
wrapper there so every test gets it for free. The June 2026 RDS
rollout showed the symptom in `DashboardPage.pendingInvitationRedirect.test.tsx`
which uses `render()` from `@testing-library/react` directly. Either
swap to `renderWithProviders` (if it exists in the same repo) or
inline the wrapper in that one test.

### Detection recipe (under 30 seconds)

When a test that has been passing for weeks starts failing with
`No QueryClient set, use QueryClientProvider to set one`:

1. Run `git log --since=1.month.ago --oneline -- src/<testfile>` to
   find the last test edit.
2. Run `git log --since=1.month.ago --oneline -- src/<page file>`
   to find the last page edit.
3. The diff between the two is where the new component was
   added to the page without a test update. Open the page's
   imports; find the new component; check if it uses
   `useQuery`/`useAppQuery`.
4. Apply Option A (mock the barrel) if the new component uses
   3+ hooks. Apply Option B (QueryClientProvider) if the test
   needs the data.

### Anti-patterns to avoid

- **Don't** "fix" by silently skipping the new component with
  `if (someCondition) return null`. The condition will drift and
  the test will lose coverage silently.
- **Don't** add `vi.mock("react-query", () => ({ ... }))` with a
  blanket mock. That breaks any test that DOES care about the
  query data, and doesn't help the failing test in any new way
  (the test would still need the specific hooks stubbed).
- **Don't** create a global QueryClient in a test setup file. Per-test
  fresh `QueryClient` instances prevent cache pollution between
  tests in the same file.

## vitest 4 — `vi.fn<T>(impl)` uses a SINGLE type argument

Pitfall surfaced June 30 2026 writing
`app_web/src/services/axios-service.client-timezone.test.ts` while
practising the vi.hoisted mock pattern. The LSP rejected this
with `Expected 0-1 type arguments, but got 2. [2558]`:

```ts
// ❌ FAILS — vitest 4 changed vi.fn to take ONE type arg, not two.
// The two-arg shape was the vitest 3 / jest signature.
const clientTimezoneServiceMock = {
  get: vi.fn<string | null, []>(() => "America/Santiago"),
};
```

```ts
// ✅ WORKS — either drop the type arg (TS infers from impl)
const clientTimezoneServiceMock = {
  get: vi.fn((): string | null => "America/Santiago"),
};

// ✅ OR keep the return-type as a single generic
const clientTimezoneServiceMock = {
  get: vi.fn<() => string | null>(() => "America/Santiago"),
};
```

Why the breaking change: vitest 4 unified the type signature to
mirror modern Jest types. The old `<TReturn, TArgs>` shape made
`TArgs` an awkward tuple you almost always wrote as `[]`, and the
inference rarely helped. The new `<TReturn>` (or just plain
`vi.fn(impl)`) is shorter and TS infers the args from `impl`'s
own signature.

Detection: if a test patch fails LSP with
`Expected 0-1 type arguments, but got 2.` at a `vi.fn<...>(...)`
call, that's the trap. Strip the args tuple, optionally keep the
return type, done.

**Also affected**: `vi.fn<TReturn, TArgs>` signatures in any code
copied from vitest 3 tutorials (the old form was also valid Jest
syntax). The pattern is common in `vi.hoisted(() => ({...}))`
mock-factory blocks in `app_web` test files that use the
`axios.create` capture pattern.

## When this skill does NOT apply

- **Jest tests with `jest.mock`**: the same pattern applies but
  the syntax differs (`jest.mock("@/module", () => ({…}))`). The
  fix is structurally identical. Load this skill for the
  *pattern*; adapt the syntax locally.
- **`vi.mock` for a module that has only one import style in
  the entire codebase**: the dual-export requirement is
  trivially satisfied. No special handling needed.
- **Mocks that need to return different shapes for different
  import sites**: rare, but possible if the default and named
  exports intentionally differ (e.g. a module with a default
  function and a named constant). In that case, point them to
  different implementations in the factory.

## Verification

After applying the fix:

```bash
# Focused first (fast feedback)
npx vitest run <path-to-failing-test>

# Then full suite to catch any cross-test pollution
npx vitest run

# Typecheck (in case the mock shape needs to match a TS type)
npx tsc --noEmit

# Whitespace
git diff --check
```

If vitest shows a different error after the fix, the original
error was masking a deeper issue (e.g. a `vi.fn()` not returning
what the SUT expects). Re-read the new error before re-trying.
