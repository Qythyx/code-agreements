# Code Agreements — General

Language-agnostic guidance that applies to every project. See [`README.md`](README.md) for how these
files are organised and maintained.

## Design

### Favour readability over performance

Optimise for the person reading the code next. Reach for a faster-but-harder-to-follow version only
when there's a measured reason to.

### Don't repeat yourself

A fact should be stated in one place. Duplication is a maintenance liability before it's anything
else, because the copies drift.

### Centralise a repeated correctness check behind one named guard

Duplicating a check is worse than duplicating a fact: every hand-written copy is a fresh chance to
invert the comparison, forget the throw, or pick the wrong status code. Where the check enforces a
security invariant, a bad copy is a vulnerability rather than drift.

```csharp
// Before: written out at seven call sites
if (await ResolveAccountID(credentials) != credentials.AccountID)
{
    throw new DatabaseException(HttpStatusCode.Forbidden, "Invalid credentials for the requested account");
}

// After
await AssertCredentials(credentials);
```

### Avoid variables that are used only once

Inline the expression unless naming it genuinely aids reading or improves the formatting.

### Reuse common UX components rather than styling each page

Identify the fundamental components and give them consistent styles. Define a unique style only when
the UX is genuinely unique — otherwise every page becomes its own small design system.

### Avoid hardcoded values, especially duplicated ones

Two literals that are the same value are one constant waiting to be extracted. Define the constant
and use it.

### Keep only genuinely-mutable values in component state

State is for what changes in response to interaction. A value set once when the component mounts, or
derived from data the component already holds, is not state — compute it at render time (inlined if
it's used once). Storing it lets the copy drift from its source and pads the state object with
fields that never move.

## Naming

### Name an identifier parameter for what it identifies

A bare `id` forces the reader to infer whose id it is, and the enclosing method name is not enough
to disambiguate — a method touching two entities leaves it an open question which one the id
references. Say it in the name.

```csharp
// `id` could plausibly be the order's or the payment's
Task<OrderViewWithAccount> ReconcileOrderPaymentAsync(string id)       ->
Task<OrderViewWithAccount> ReconcileOrderPaymentAsync(string orderID)
```

### Name a lookup for what it returns, not for what a caller might check

A lookup called `Verify*` reads as though it enforces something, so a caller may use it as a guard
and forget the comparison it was supposed to make. Name it for the value it produces, and let a
separate `Assert*` do the enforcing.

```csharp
VerifyCredentials  ->  ResolveAccountID   // returns an account ID, or null
                       AssertCredentials  // throws when the caller doesn't own the account
```

### Name a wire type for its role in the exchange

`Request` and `Response` say which direction a type travels and what it's for. `Message` says only
that it crosses the wire, which every one of them does.

```csharp
SetOfferQuantityMessage   ->  SetOfferQuantityRequest
CardSetupSessionResponse  ->  CreateCardSetupSessionResponse
```

## Style

### Always use curly braces for code blocks

Even for single-line statements. The exception is React code, where
`onMouseDown={e => e.stopPropagation()}` is fine.

### Use early returns

Handle the exit case and get out, rather than nesting the main path inside a conditional.

## Comments

### Only comment what the code can't say

Comment when the code is genuinely tricky or unintuitive, or when the codebase requires
documentation. Don't restate what the next line does — clear names and structure do that job better.
A self-evident guard needs no comment above it: an `if` whose condition and thrown message already
say what's being rejected explains itself, and narrating the consequence of _not_ guarding is
rationale that belongs in the commit, not the code (see "Keep rationale in the commit").

```csharp
// Before — a paragraph explaining a guard that already speaks for itself
// A negative quantity would survive the clamp below (it is the minimum), inflating stock
// via a positive beverage delta and storing a negative order entry. Reject it outright.
if (quantity < 0)
{
    throw new DatabaseException(HttpStatusCode.BadRequest, "Quantity cannot be negative");
}

// After — the condition and message are the whole story
if (quantity < 0)
{
    throw new DatabaseException(HttpStatusCode.BadRequest, "Quantity cannot be negative");
}
```

### A comment's stated reason must be a scenario that can actually occur

Before writing (or keeping) a comment that justifies code with a scenario, check the scenario is
reachable in this system. A plausible-but-false reason is worse than none: it survives review and
sends the next reader investigating the wrong thing. If no reachable scenario can be named, question
the code instead of the comment.

```csharp
// Before — "legacy customers" cannot exist in a service that hasn't launched
// ...falling back to the most recently added card if no default is set (legacy customers).

// After — the cause that can actually happen
// ...falling back to the most recently added card if no default is set — possible when card setup
// succeeded on Stripe's side (card attached) but the completion handler that records the default
// never ran, e.g. the customer closed the browser before returning to the success URL.
```

### Don't comment on what used to be there

A comment explaining what was removed, what this replaces, or what is no longer needed is written
for the reviewer, not the next reader. It's noise the moment the change merges.

### Keep rationale in the commit, not the comment

Why an approach was chosen over an alternative, or how a dependency's internals force the current
shape, is not "what the code can't say" — it's context for whoever reviews the change. It outlives
the context that justified it and rots in place, and it drags in names of things the code no longer
references. State the local intent the line can't; put the "why not the other way" in the commit or
PR.

```js
// Before — names the rejected alternative and walks the dependency's internals
// ...preserving config that `nbgv set-version` would strip. Then stage it ourselves:
// @semantic-release/git runs from this cwd and builds its list with `git ls-files`,
// which never reaches ../../version.json...
prepareCmd: 'node scripts/bump-version-json.mjs ${v} && git add ../../version.json',

// After — only the coupling this line can't show on its own
// Bump version.json (repo root) and stage it for the release commit below.
prepareCmd: 'node scripts/bump-version-json.mjs ${v} && git add ../../version.json',
```

## Warnings

### Never silence a warning to make it go away

Fix the underlying issue. If it genuinely can't be fixed reasonably, ask before suppressing it —
don't decide unilaterally that a warning is acceptable.

## Encapsulation

### A consumer must not know its provider's internal structure

If a class exists to expose something in a controlled way, callers go through it — they don't reach
around it. A caller that knows about folders, file extensions, or storage layout is doing the
provider's job.

```csharp
// Wrong — the caller knows sprocs are files, in a folder, with an extension
foreach (var filename in EmbeddedResources.ListFilenames(EmbeddedResources.StoredProceduresFolder))
{
    Id = Path.GetFileNameWithoutExtension(filename),
    Body = EmbeddedResources.Read(EmbeddedResources.StoredProceduresFolder, filename),
}

// Right — it asks for what it wants and knows nothing about how it's stored
foreach (var procedure in Enum.GetValues<StoredProcedure>())
{
    Id = procedure.ToString(),
    Body = EmbeddedResources.GetStoredProcedure(procedure),
}
```

### Prefer a closed, typed API over an open-ended one

A public method taking arbitrary strings is a door anyone can walk through with anything. Take a
value from a known set instead, and make the string-taking version private.

### Put mapping logic in the class that owns the thing being mapped

Not in the caller, and not in a satellite helper.

```csharp
// Before: a satellite helper formats the payment method
public static class CardDisplay
{
    public static string? ForCard(RegisteredPaymentMethod pm) => ...;
}

// After: the type that holds the data formats itself
public record StripeAccountInfo(string CustomerID, string? Brand, string? Last4)
{
    public string Describe() => ...;
}
```

### Don't expose an identifier every caller converts anyway

If each call site converts the identifier to the thing on the same line, the identifier is ceremony.

```csharp
// Before: all 8 call sites read GetTemplate(DefinedTemplates.X)
public const string WelcomeEmail = "WelcomeEmail.html";

// After: the caller asks for a template and gets a template
public static string WelcomeEmail => Read("WelcomeEmail.html");
```

## Simplicity

### Complexity must justify itself against a concrete, likely failure

Not a theoretical one. If the failure it guards is narrow, self-correcting, or already handled a
layer down, the guard is not worth its weight.

### Don't issue a write that only re-asserts state the server already holds

When the server has already applied a change out of band, read the updated state back rather than
sending an update that sets it to what it already is. The redundant write is a wasted round trip and
a second chance to fail. (After the card-setup return switched the account server-side, the app was
calling UpdateAccount to set the same thing; a refresh replaced it.)

### Don't defend against a failure the codebase ignores elsewhere

If nothing else re-creates a deleted database item, a special case for one resource type is
inconsistent, not careful. Consistency with the surrounding code is itself an argument.

### Prefer removing a hazard to documenting it

When a trap exists only because something is inconsistent, fix the inconsistency instead of writing
the warning. A documented hazard still catches people, and the document rots independently of the
thing it describes.

```
Four test projects, three different AssemblyName patterns; two resolved
differently in Debug and Release.

Documenting it:  "watch out — the assembly name changes per configuration"
Removing it:     delete the overrides; all four resolve to the project name
```

### Prefer one general rule over several specific ones

A rule per instance is a list someone has to maintain, and lists rot.

```xml
<!-- Instead of one entry per file, or one per folder -->
<EmbeddedResource Include="Resources/*/*" />
```

## Tests

### A test asserts its marginal behavior, not a copy of another test's expectations

When a second test exists only to pin one additional behavior (an error propagates, a status passes
through), match the collaborator loosely and assert just that behavior. Duplicating another test's
exact-match setup adds a second copy that must be edited in lockstep while verifying nothing the
first test doesn't already cover — and a test whose only marginal assertion is default behavior
(e.g. an exception propagating through code with no try/catch) may not be worth keeping at all.

## Evidence

### Prefer established, documented practice over an invented pattern

When proposing something novel, check whether it's actually an established practice, and say plainly
when it isn't. A pattern transferred from another domain is a reasoned guess, not received wisdom —
present it as one. An unusual pattern also costs future readers, who won't recognise it and have no
community experience to draw on.

### Prove a guard can fail

A test that cannot fail is not protection. Deliberately break the thing it guards and confirm it
goes red before claiming it works.

## Reading the situation

### Asymmetry is a smell

When similar things are treated differently for no stated reason, the difference is usually neglect
rather than intent, and is worth pulling on. (Two folders declared and a third not; ten copy rules
for seven files.)
