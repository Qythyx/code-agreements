# Code Agreements — TypeScript and React

TypeScript, React, and web front-end guidance that applies to every project. See
[`README.md`](README.md) for how these files are organised and maintained.

## Functions

### Use `const` arrow functions instead of function declarations

```ts
export const isDateRangeValid = (begins: string, ends: string): boolean =>
	new Date(begins).getTime() < new Date(ends).getTime();
```

### Name event handlers with a `handle` prefix

Use `handleClick` for `onClick`, and `handleKeyDown` for `onKeyDown`.

## Types and visibility

### Give class members explicit visibility

Make a member public only when something outside the class needs it.

## Forms

### Trim all strings in a form payload once, at save

Pass the whole form-data object through one recursive trim helper before building the request,
instead of calling `.trim()` per field.

## Layout

### Don't hardcode sizes

Let controls grow and shrink to fit the available space and their content.

## Accessibility

### Make interactive elements accessible

Give each one `tabindex="0"`, an `aria-label`, and a keyboard handler alongside the click handler.

## Testing

### In a mock assertion, pin the arguments under test and match the rest

Write `toHaveBeenCalledWith(expect.objectContaining({ … }), expect.anything())`, so adding a
parameter doesn't break tests that have nothing to do with it.
