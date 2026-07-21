# Code Agreements — C#

C# and .NET guidance that applies to every project. See [`README.md`](README.md) for how these files
are organised and maintained.

## Structure

### Organise code by feature or domain, not by type

Group the related model, service, and viewmodel together rather than filing each by its kind.

### Make namespaces reflect the folder structure

### Favour composition over inheritance

### Nest a type that serves only as one interface's contract inside that interface

A result or DTO used only as the input or output of a single interface belongs nested inside it —
not in a sibling file, not at namespace scope. A type used only by one class nests privately inside
that class the same way. It scopes the name to what it serves and keeps the folder from filling with
one-line files.

Nesting a public type _signals_ the coupling but doesn't _enforce_ it: it's still constructible as
`IFoo.Bar`, and `internal` isn't an option when the implementation lives in a different assembly.
Call sites reference it as `IFoo.Bar`, or alias it per file (`using Bar = …IFoo.Bar;`) where that
reads noisily.

```csharp
// Before: ChargeResult.cs, CardSetupSession.cs — each a one-line sibling of IPaymentService
public record ChargeResult(PaymentMethodStatus Status, string? PaymentIntentID, string? DeclineCode);

// After: nested in the interface whose methods return it
public interface IPaymentService
{
    Task<ChargeResult> ChargeCustomerAsync(...);
    record ChargeResult(PaymentMethodStatus Status, string? PaymentIntentID, string? DeclineCode);
}
```

## Naming

### Don't prefix a type with `Defined`

Name it for what it is.

```csharp
DefinedStoredProcedures  ->  StoredProcedures
DefinedTemplates         ->  HtmlTemplates
```

### Prefer explicit qualification over `using static`

The extra token at the call site is worth the reader knowing where a member came from.

```csharp
// Preferred
using MyApp.Core.Resources;
EmbeddedResources.GetHtmlTemplate(HtmlTemplates.WelcomeEmailCss)

// Avoid
using static MyApp.Core.Resources.EmbeddedResources;
GetHtmlTemplate(HtmlTemplates.WelcomeEmailCss)
```

### A new namespace shadows a using-imported class of the same name

Namespaces win over using-imported types in C# name resolution, so introducing `Foo.Bar.Templates`
silently breaks callers of a `Templates` class imported from elsewhere. Check for a type of that
name before naming a namespace.

## Style

### Omit redundant access modifiers on interface members

Interface members — including nested types — are public by default; don't spell out `public`.

```csharp
public interface IAccountManager
{
    // not: public sealed record AccountLookupResult(...)
    sealed record AccountLookupResult(AccountLookupStatus Status, Account? Account);
}
```

### Use PascalCase for public members, camelCase for private fields

### Mark classes `sealed` unless inheritance is required

### Use `readonly` for fields that aren't reassigned

### Prefer records and immutable types for data objects

### Use expression-bodied members for simple properties and methods

### Prefer `var` when the type is obvious from the right-hand side

Otherwise use the explicit type.

### Use object and collection initialisers

### Use pattern matching and switch expressions for complex branching

## Functions

### Avoid private functions with a single call site

Inline them unless naming genuinely aids reading. Also consider defining the function within another
function when it is only used there; this prevents it from showing up in intellisense in outside
contexts.

### Make every argument explicit at the call site

Three rules, one intent — the signature should say what it takes, and the call should say what it
passes:

- Avoid named arguments unless the language requires them.
- Avoid arguments with default values; make the caller state them.
- Avoid nullable arguments.

## Types and contracts

### Prefer an enum over a magic string

When a value names one of a known set — especially when that identity crosses a contract boundary.
The enum removes the string and gives the compiler something to check.

```csharp
// Before
Task<T> ExecuteStoredProcedure<T>(string storedProcedure, dynamic[] parameters);
Database.ExecuteStoredProcedure<Customer>(StoredProcedures.CreateOrUpdateCustomer, [...]);

// After
Task<T> ExecuteStoredProcedure<T>(StoredProcedure storedProcedure, dynamic[] parameters);
Database.ExecuteStoredProcedure<Customer>(StoredProcedure.CreateOrUpdateCustomer, [...]);
```

## Enums and switch expressions

### Declare explicit enum values

Roslynator enforces this (`RCS1161`).

```csharp
public enum StoredProcedure
{
    CreateOrUpdateCustomer = 0,
}
```

### Keep the discard arm on a switch expression over an enum

Dropping it does _not_ buy compile-time exhaustiveness: covering every named value still leaves
**`CS8524`** ("does not handle some values … involving an unnamed enum value"), because an enum can
hold any underlying int. `CS8509` only fires for missing _named_ values, and you cannot get it
without also getting `CS8524`. Keep `_ => throw new ArgumentOutOfRangeException(nameof(x))` and
cover the forgotten-member case with a test that iterates `Enum.GetValues<T>()`.

### `ArgumentOutOfRangeException(string)` takes `paramName`, not a message

Pass `nameof(x)`; passing the value reports `(Parameter 'OrderClosing')`.

## Strings

### Don't spell out `StringComparison` to compare two strings for equality

`==` and `!=` on two `string`s already compare ordinally and handle null, so the explicit form is
longer without behaving differently. On a security-critical line the smaller diff is worth more,
because it's the one a reviewer has to read.

```csharp
!string.Equals(a, b, StringComparison.Ordinal)   // same result
a != b                                            // prefer
```

### Derive a string from the semantically correct source

Rather than hardcoding it or re-deriving it from something that merely happens to match.

```csharp
// Resource names come from the root namespace + folder path, so read the namespace —
// not the assembly name, which is equal only by coincidence.
var name = $"{typeof(EmbeddedResources).Namespace}.{folder}.{filename}";
```

## Error handling

### Catch only specific exceptions you can handle

Avoid catching `Exception` unless rethrowing or logging. Use exceptions for exceptional conditions,
not for control flow.

### Use custom exception types for domain-specific errors

## Dependency injection

### Use constructor injection, not a service locator

Favour interfaces and abstractions at the boundaries so the thing can be tested.

## Async

### Use `async`/`await` for I/O operations

Return `Task` or `Task<T>`; don't block on async code.

### Avoid `async void` except for event handlers

### Use `ConfigureAwait(false)` in library code that isn't UI-bound

## Documentation

### Document every parameter on a public API

XML doc comments on public APIs get a `<param>` tag for each parameter — no exceptions. Document
assumptions, side effects, and non-obvious design decisions; don't restate what the signature
already says.
