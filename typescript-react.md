# Code Agreements — TypeScript and React

TypeScript, React, and web front-end guidance that applies to every project. See
[`README.md`](README.md) for how these files are organised and maintained.

## Functions

### Always specify return types, including on lambdas

### Use `const` arrow functions instead of function declarations

Define a type for it where one is possible.

### Name event handlers with a `handle` prefix

`handleClick` for `onClick`, `handleKeyDown` for `onKeyDown`.

## Documentation

### No JSDoc `@returns` on a function that returns void

## Types and visibility

### Give class members explicit visibility

Prefer not to expose a function as public unless something outside the class actually needs it.

## Forms

### Trim user-entered text at save with one blanket deep-trim of the payload

One general rule instead of a `.trim()` per field: pass the whole form-data object through a
recursive trim-all-strings helper (`utils/deepTrimStrings.ts`) just before building the request.
Per-field trimming invites the asymmetry where only some fields are trimmed. Fields that need
empty→undefined mapping on top (nullable wire fields) keep their explicit handling.

## Layout

### Don't hardcode sizes

Controls should grow and shrink to fit the available space and their content.

### Restoring a scroll or pan/zoom position across a data change needs an identical layout, or an anchor

Reapplying the same offset only shows the same content if the DOM lays out identically. When the data
varies — different-sized rows, an optional panel, a wider selected item — the same transform lands
somewhere else. Make the varying pieces a uniform size, or pin to a specific element's on-screen
position, instead of trusting the raw offset.

## Accessibility

### Make interactive elements accessible

An interactive element needs `tabindex="0"`, an `aria-label`, and a keyboard handler alongside its
click handler — not just the mouse path.

## Hooks

### Extend a tuple-returning hook by appending, never by repurposing a position

A tuple's positions are its contract, and reordering one while the types stay the same is invisible to
both the compiler and the reader — every existing destructure silently starts meaning something else.
Append the new element even when another order reads better in isolation.

```ts
// useUrlState returns [value, setValue]. The debounced variant appends, so an existing
// `const [searchTerm, setSearchTerm] = useUrlState(...)` keeps meaning what it meant.
const [searchTerm, setSearchTerm, debouncedSearchTerm] = useDebouncedUrlState(
  "search",
  "",
  SearchDebounceMs,
);

// Leading with the debounced value would compile everywhere and silently make every
// input bound to position 0 lag by the debounce interval.
```

## Testing

### Assert the arguments a test cares about and match the rest

`toHaveBeenCalledWith` compares the whole argument list, so an assertion that spells out every
parameter fails the moment one is added — in a file that otherwise has nothing to do with the change.
Pin the arguments under test and let a matcher absorb the others.

```ts
// Adding an AbortSignal parameter broke this assertion in a test about role filtering
expect(service.GetUsers).toHaveBeenCalledWith(
  expect.objectContaining({ RoleFilter: UserRole.Pending }),
);

// After
expect(service.GetUsers).toHaveBeenCalledWith(
  expect.objectContaining({ RoleFilter: UserRole.Pending }),
  expect.anything(),
);
```

### A green test run and a green lint do not mean the types check

Vitest transpiles rather than type-checks, and ESLint's type-aware rules cover far less than the
compiler. A change can therefore break the build while the whole suite and the linter stay green.
Run `tsc --noEmit` (or the production build) before believing a refactor landed.

### Verify pointer interactions with real pointer events, not `element.click()`

A synthetic `element.click()` skips the pointerdown/move/up sequence, so it sails past the machinery a
real click goes through — pointer capture, hit-testing, drag-vs-click discrimination — and can pass
while the real interaction is broken. Drive the check through real events: a Playwright locator
click, or dispatched `PointerEvent`s.
