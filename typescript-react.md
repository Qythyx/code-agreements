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

## Layout

### Don't hardcode sizes

Controls should grow and shrink to fit the available space and their content.

## Accessibility

### Make interactive elements accessible

An interactive element needs `tabindex="0"`, an `aria-label`, and a keyboard handler alongside its
click handler — not just the mouse path.
