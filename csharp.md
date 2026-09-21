# Code Agreements — C\#

C# and .NET guidance that applies to every project. See [`README.md`](README.md) for how these files
are organised and maintained.

## Structure

### Put a shared helper on the class that owns the concept, not on a common base class

Don't move a helper to the shared base class because a sibling class needs it; every other subclass
then inherits something it never calls. Make it a `static` on the owning class, or expose it on the
owner's interface and inject the owner. Don't create a new class only to share it.

### Nest a type inside its only user

Declare a result or DTO inside the interface when only that interface uses it, and nest a type
privately in the class when only that class uses it. Move it to a shared place only when a second
real caller needs it.

## Style

### Use a primary-constructor parameter directly

Don't copy it into a `private readonly` field. Add a field only when it adds something, such as a
different type or a computed value.

### Add a `using` directive instead of writing a fully qualified type name

Add the `using` line even when the code around it writes the same type fully qualified. Shorten
those existing uses in the same file too; the style checker reports them once the `using` exists.
Write the fully qualified name only when the `using` would cause a name conflict.

### Mark classes `sealed` unless inheritance is required

### Group members with `#region`, not comment banners

Write `#region Name` … `#endregion Name`. Add a region only where a class has real clusters of
members; don't wrap a few adjacent one-liners.

### Put an `#if` around only the part that varies

Do this even in the middle of an argument list. Don't write the whole expression twice in
`#if`/`#else`.

```csharp
Label(AppResources.SettingsLanguage
#if DEBUG
        , TapGestureRecognizer(HandleHiddenTestModeTapped)
#endif
    ).ThemeHeader(),
```

## Functions

### Prefer a local function for a helper only one function uses

Inline the helper first if you can (see "Prefer inlining…" in general.md). Make it a local function
inside its caller, not a class member, when a name helps.

### Make every argument explicit at the call site

Avoid named arguments, and avoid defaulted parameters.

### Don't make a parameter nullable for one caller

Handle null with `??` at the call site when only one call site can have null. Make the parameter
nullable only when null is part of the method's meaning for every caller.

## Strings

### Prefer `string.Empty` over `""` in production code

Tests may use `""`.

### Compare strings for equality with `==`

Don't write `string.Equals(a, b, StringComparison.Ordinal)`; `==` is already ordinal and null-safe.

## Error handling

### Use the `ThrowIf*` helpers for argument checks

For example `ArgumentOutOfRangeException.ThrowIfNegative(x)` and
`ArgumentException.ThrowIfNullOrWhiteSpace(s)`. Use two calls for a range, because there is no range
helper.

### Return an expected outcome; throw only for the unexpected

Put an outcome in the return type, as an enum or a result, when the caller would otherwise catch it
on the next line. Throw for what the caller can't reasonably expect, such as an id that doesn't
exist.

## Dependency injection

### Use constructor injection, not a service locator

### Don't add an interface for a class with one implementation and one caller in the same layer

Add an interface only where something is really substituted: a lower layer depending on an
implementation that an outer layer supplies.

## Async

### Use `ConfigureAwait(false)` in library code that isn't UI-bound

## Tests

### Make one claim per test

Split a test whose name contains "And" into two. Use an `AssertionScope` only when one claim needs
several assertions.

### Keep all tests for a production class in one file

Name it `FooTest.cs` for `Foo`. Group themed subsets as nested `sealed` classes extending the test
base, not as extra files or `#region`s.

## Documentation

### Document every public API

State what the class or method does so that it would be understandable to someone reading it fresh.
Add a `<param>` for each argument. Document assumptions and side effects; don't restate the
signature.

### Start a doc summary with a verb that says what the member is for

Write "Checks that…" or "Fetches…", not "Throws unless…" or "Returns…". Use only nouns that appear
in the signature, or in the project's glossary if it has one. Leave out a caller's reasons, the
history, and how the member works inside; put specifics in the `<param>` and `<returns>` tags. Split
the method when its summary can't be written plainly.

### Name code in a doc comment with `<see cref>`, not `<c>`

Use `cref` because the compiler checks it, so a rename flags it. Use `<c>` only for what the
compiler can't resolve: shell commands, other languages, another class's private members.
