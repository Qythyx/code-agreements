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

### Un-nest a type the moment a second owner has to name it

A nested type is only nameable by code that can reference its container. When something in a lower
layer needs it too, nesting forces that consumer to take a weaker type — a base type, or `object` —
purely to dodge a reference it can't have. Move the type to the shared layer both can see, as a
top-level type named for what it is. The nesting rule above assumes a single owner; a second one
retires it.

```csharp
// Before: nested in the app-layer parser, so App.Core — which cannot reference the app — had to
// weaken its parameter to `Enum` to accept it
public static class NotificationParser { public enum ParseResult { … } }
public void TrackNotificationParseFailed(Enum result)

// After: the enum moves down to App.Core, and both ends are strongly typed
public enum NotificationParseResult { … }
public void TrackNotificationParseFailed(NotificationParseResult result)
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

### Don't namespace-qualify a name that already resolves, including in `<see cref>`

A namespace prefix on a type that a `using` or the enclosing namespace already brings into scope is
noise — the reader gains nothing the compiler didn't already know. This is about namespace prefixes,
not about `using static`: qualifying a member by its own type still earns its token, per the entry
above.

```csharp
// Before
foreach (var template in Enum.GetValues<Core.Resources.HtmlTemplate>())
public class WebAuthenticationCallbackActivity : Microsoft.Maui.Authentication.WebAuthenticatorCallbackActivity
/// Mirrors the framework's <see cref="MobileJourneys.Dsl"/>.

// After
foreach (var template in Enum.GetValues<HtmlTemplate>())
public class WebAuthenticationCallbackActivity : WebAuthenticatorCallbackActivity
/// Mirrors the framework's <see cref="Dsl"/>.
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

### Don't alias a primary-constructor parameter with a private field

A primary-constructor parameter is in scope for the whole class body, so a `private readonly` field
initialised from it is a second name for one thing — the same objection as a property that only
forwards. Use the parameter directly.

```csharp
// Before
public partial class PostalCodeManager(..., IHttpClient HttpClient)
{
    private readonly IHttpClient _httpClient = HttpClient;
    ... await _httpClient.GetByteArrayAsync(DownloadUrl);
}

// After
    ... await HttpClient.GetByteArrayAsync(DownloadUrl);
```

The field is still warranted when it adds something the parameter can't: a different type, a
computed value, or an initialiser the parameter doesn't carry.

### Use PascalCase for public members, camelCase for private fields

### Mark classes `sealed` unless inheritance is required

### Use `readonly` for fields that aren't reassigned

### Prefer records and immutable types for data objects

### Use expression-bodied members for simple properties and methods

### Prefer `var` when the type is obvious from the right-hand side

Otherwise use the explicit type.

### Use object and collection initialisers

### Use pattern matching and switch expressions for complex branching

### Group members with `#region` blocks, not comment banners

When a class has natural clusters of members, delimit them with `#region Name` / `#endregion Name`
rather than a `// --- Name ---` comment. Name the `#endregion` too, so the closing tag says which
region it ends. The regions collapse in the editor; a comment banner doesn't.

A region has to earn its place: a handful of adjacent one-line constants is already legible, and
wrapping it costs two lines to save none.

```csharp
// Before          // After
// --- Actions --- #region Actions
                   …
                   #endregion Actions
```

## Conditional compilation

### Wrap an `#if` around the varying fragment, not around two copies of the expression

When only part of an expression is conditional, put the directive inside it — even
mid-argument-list. An `#if`/`#else` holding two near-identical copies makes the reader diff them to
find the difference, and every later edit has to be made twice. csharpier leaves the inline form
alone.

```csharp
// Before: two copies, one differing argument
#if DEBUG || MOCK
    Label(AppResources.SettingsLanguage, TapGestureRecognizer(HandleHiddenTestModeTapped)).ThemeHeader(),
#else
    Label(AppResources.SettingsLanguage).ThemeHeader(),
#endif

// After: the directive scopes the one thing that varies
    Label(AppResources.SettingsLanguage
#if DEBUG || MOCK
            , TapGestureRecognizer(HandleHiddenTestModeTapped)
#endif
        ).ThemeHeader(),
```

## Functions

### Prefer a local function for a helper only one function uses

Defining it inside its caller puts it where its only reader is and keeps it out of intellisense in
outside contexts.

### Have an `Ensure*` validation helper return its subject

Returning the validated value lets the caller chain instead of validating on one line and using on
the next.

```csharp
// Before
await EnsureBeverageSupplierAndSkuValidAsync(beverage);
return await Database.CreateItemAsync(beverage);

// After
return await Database.CreateItemAsync(await EnsureBeverageSupplierAndSkuValidAsync(beverage));
```

### Return `Task`, not `Task<T>`, when no caller uses the value

A count or flag that only the method itself consumes — for a log line, say — turns every call site
into `_ = await …`. Drop the result and the discard goes with it. The same reasoning as a
`Task<bool>` that only ever returns `true`.

### Make every argument explicit at the call site

Three rules, one intent — the signature should say what it takes, and the call should say what it
passes:

- Avoid named arguments unless the language requires them.
- Avoid arguments with default values; make the caller state them.
- Avoid nullable arguments.

### Coalesce at the one call site that can pass null, rather than widening the parameter

A nullable parameter announces that _every_ caller may pass null, so the function and each future
call site inherit the question. When only one call site actually has a null to hand, absorb it there
and keep the signature honest.

```csharp
// Before: one iOS caller could pass null, so the parameter took it on behalf of all six call sites
public static bool TryParse(string? json, out INotification notification)

// After: the ambiguity stays where it lives
public static bool TryParse(string json, out INotification notification)
NotificationParser.TryParse(userInfo[FieldJson]?.ToString() ?? string.Empty, out var notification)
```

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

### Validate a record's positional property in the initializer, not an init accessor

Redeclare the property with a validating initializer expression. A property initializer assigns the
backing field directly and does **not** run a custom `init` accessor, so validation placed in the
accessor is silently skipped on exactly the paths that matter — the primary constructor, and
System.Text.Json deserialization, which constructs records through it. The accessor would only fire
for an external `with { … }`. Because `ArgumentOutOfRangeException.ThrowIf*` returns `void`, it
can't be the initializer expression directly; use a ternary, or a static helper the initializer
calls.

```csharp
// Runs on every path, including deserialization
public record SetOfferQuantityMessage(string AccountID, string OfferID, int Quantity) : IMessage
{
    public int Quantity { get; init; } =
        Quantity >= 0 ? Quantity : throw new ArgumentOutOfRangeException(nameof(Quantity), "Quantity cannot be negative");
}

// Trap: the `= Quantity` initializer bypasses this accessor, so the guard never runs on construct/deserialize
public int Quantity { get; init { ArgumentOutOfRangeException.ThrowIfNegative(value); field = value; } } = Quantity;
```

### A non-nullable record property is a compile-time claim, not a runtime guarantee

System.Text.Json constructs records through the primary constructor and passes null for any JSON
property that is missing or explicitly null — the non-nullable annotation does not stop it. If
documents or payloads can omit the property, either coerce in a property initializer
(`public string SKU { get; init; } = SKU ?? string.Empty;`) or guarantee the data is clean (a
migration that backfills every document). Deciding which is a judgment call: the coercion was
dropped on Beverage/Supplier because a pre-launch migration guaranteed the data instead.

### `null!` on the way out is a lie a strict serializer catches at runtime

The null-forgiving operator silences the compiler; it does not make the value non-null. Where the
serializer respects nullable annotations it refuses to _write_ null from a non-nullable member, so
the throw lands at send time — and if the send is wrapped in a catch that reports transport failure,
the request never leaves and the symptom is "the network is down".

Pass the empty string instead when the member genuinely has no value yet, per the
`Prefer empty string over null for optional text` entry in `general.md`.

```csharp
// Before: the caller is asking who it is, so it cannot know the ID — but null cannot be serialized
new CredentialsMessage(new(email, null!, clientToken))

// After
new CredentialsMessage(new(email, string.Empty, clientToken))
```

### Turn "this must go through that path" into a compile error with an `[Obsolete(error: true)]` overload

C# has no negative generic constraint — you cannot write `where T : not IFoo`. To stop a caller
routing a marked type through the general-purpose method, add an overload taking the more-derived
type and mark it as an error. Overload resolution prefers it whenever the argument's static type is
the marked one, so the wrong call fails to compile. It catches the ordinary mistake, not a value
upcast to the base interface first.

```csharp
[Obsolete("Authenticated requests must go through SendAuthenticatedMessage.", error: true)]
[SuppressMessage("Style", "IDE0051", Justification = "Exists to make the wrong call fail to compile.")]
private static Task<ServiceResult<T>> SendMessage<T>(string path, IAuthenticatedMessage message) =>
    throw new InvalidOperationException();
```

Keep the shared implementation in a separate core method that both the guarded and the marked path
call, rather than casting past the guard at the one legitimate call site.

### Expose a private collection as a snapshot, not as a live view of it

Handing back the backing collection — or a deferred LINQ query over it, which is the same thing
evaluated later — gives the caller a handle on your state instead of a value. They can cast an
`IEnumerable<T>` back to `List<T>` and write to it, and a caller enumerating while you mutate gets a
moving target or a "Collection was modified" throw. Copy before returning, inside the lock if there
is one. A read-only return type declares the intent but doesn't enforce it; the copy is what does.

```csharp
// Before: deferred over the private collection, so the caller evaluates it against live state
return Task.FromResult(sorted.AsEnumerable());

// After. The .AsEnumerable() is load-bearing, not redundant: Task<T> is not covariant, so
// Task<T[]> will not convert to Task<IEnumerable<T>>. Don't tidy it away.
return Task.FromResult(sorted.ToArray().AsEnumerable());
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

### Guard `Enum.TryParse` with `Enum.IsDefined` when you mean "a real named member"

`Enum.TryParse` returns `true` for any numeric string (`"999"`) and for combined `[Flags]` strings,
handing back an out-of-range value that isn't a declared member. Pair it with `Enum.IsDefined` when
the question is whether the input names an actual member.

```csharp
// Accepts "999" and "A, B" as success
Enum.TryParse<T>(name, ignoreCase: true, out var value)
// Only genuine named members survive
Enum.TryParse<T>(name, ignoreCase: true, out var value) && Enum.IsDefined(value)
```

## Serialization

### A nullable wire member is optional only once it also carries a default

Under `RespectRequiredConstructorParameters`, a constructor parameter is optional exactly when it
has a default value — nullability has nothing to do with it. That matters because the codegen on the
other side usually reads optionality off the nullability alone: TypeGen exports `DateTime? From` as
`From?`, the client legitimately sends no key, and the server rejects the payload it published a
contract for. Give every nullable parameter a default, which also means putting the required
parameters first. Audit this with a test over every wire type rather than a grep, which misses
multi-line record declarations.

```csharp
// Before: TypeScript says all four are optional; the server demands all four
public record ListOrdersRequest(DateTime? From, DateTime? To, PagedRequest? PagedRequest, bool? GroupByDate) : IMessage;

// After
public record ListOrdersRequest(
    DateTime? From = null,
    DateTime? To = null,
    PagedRequest? PagedRequest = null,
    bool? GroupByDate = null
) : IMessage;
```

### A base-record constructor argument is not immune to the wire

A derived record that passes a constant to its base — `: CosmosDBDocument(SettingsID, ETag)` — looks
like it fixes that property. It doesn't. The base's positional property has an `init` accessor and
is not a parameter of the derived record's constructor, so System.Text.Json sets it from the
incoming JSON _after_ construction, and the constant is overwritten by whatever the caller sent. A
singleton document's fixed id therefore has to be enforced, not merely declared.

## Strings

### Prefer `string.Empty` over `""` in production code

Tests may use `""` literals. (Observed once: `""` arguments in src were edited to `string.Empty`
while the same-shaped test arguments were left as `""`.)

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

### Reach for the argument-exception `ThrowIf*` helpers rather than a hand-rolled throw

`ArgumentNullException`, `ArgumentException`, and `ArgumentOutOfRangeException` all carry static
guards — `ThrowIfNegative`, `ThrowIfGreaterThan`, `ThrowIfLessThan`, `ThrowIfNullOrWhiteSpace` —
that put the bound in the call and produce the standard message. There is no combined range helper;
express a range as two calls.

```csharp
// Before
private static DayOfWeek ToDayOfWeek(int closingDayOfWeek) =>
    closingDayOfWeek is >= 0 and <= 6
        ? (DayOfWeek)closingDayOfWeek
        : throw new ArgumentOutOfRangeException(nameof(closingDayOfWeek), closingDayOfWeek, "…");

// After
ArgumentOutOfRangeException.ThrowIfNegative(closingDayOfWeek, nameof(closingDayOfWeek));
ArgumentOutOfRangeException.ThrowIfGreaterThan(closingDayOfWeek, (int)DayOfWeek.Saturday, nameof(closingDayOfWeek));
```

### Catch only specific exceptions you can handle

Avoid catching `Exception` unless rethrowing or logging. Use exceptions for exceptional conditions,
not for control flow.

## Dependency injection

### Use constructor injection, not a service locator

Favour interfaces and abstractions at the boundaries so the thing can be tested.

## Async

### Use `async`/`await` for I/O operations

Return `Task` or `Task<T>`; don't block on async code.

### Avoid `async void` except for event handlers

### Use `ConfigureAwait(false)` in library code that isn't UI-bound

## Tests

### Keep all of a production class's tests in one file, grouped by nested fixtures

All tests for one production class belong in that class's single test file (`FooTest.cs` for `Foo`).
Group themed subsets as nested `sealed` fixtures extending the test base — not satellite files
(`FooProcessOrderTest.cs`), and not `#region`s. Nested fixtures are real structure where regions are
purely editorial: each group's helpers and constants stay scoped to it, the runner displays
`FooTest+ProcessOrder`, and one group can be run alone via a filter.

```csharp
// Before: AdministrationManagerProcessOrderTest.cs, AdministrationManagerListOrdersTest.cs, …

// After: one file, one class per themed group
public class AdministrationManagerTest : BaseTest
{
    // general tests…

    public sealed class ProcessOrder : BaseTest { /* charge/reconcile tests + their helpers */ }

    public sealed class ListOrders : BaseTest { /* grouped-listing tests */ }
}
```

## Documentation

### Document every parameter on a public API

XML doc comments on public APIs get a `<param>` tag for each parameter — no exceptions. Document
assumptions, side effects, and non-obvious design decisions; don't restate what the signature
already says.

### Name code in a doc comment with `<see cref>`, never `<c>`

A name inside `<c>` is a string: rename the thing and the comment silently becomes a lie. A `cref`
is resolved by the compiler, so the same rename produces CS1574 and the comment gets fixed with the
code. Qualify only as far as it takes to resolve — an ambiguous simple name needs the namespace, per
the entry above.

Leave `<c>` for the things the compiler cannot check: shell commands, JavaScript, platform constants
from outside the solution, and private members of another class.

```csharp
// Before
/// alongside <c>MobileJourneys.Dsl</c> so journey definitions read as bare calls.

// After — bare, because a using already brings it into scope
/// alongside <see cref="Dsl"/> so journey definitions read as bare calls.
```
