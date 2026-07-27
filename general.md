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

### A wire enum a not-yet-updated client reads must tolerate unknown values

A client deserializing a whole response throws the entire payload away on an enum value it doesn't
recognise, so adding a member becomes a breaking change for every already-shipped client. Read an
unknown value as a designated neutral fallback member instead of throwing, and degrade that one
field. Make the tolerance opt-in per enum — a value should never be silently mislabelled without a
deliberate choice — so an enum that only travels toward a peer that's always current (or is untrusted
input you want to reject) stays strict. The fallback has to ship in the client's _first_ release; it
can't be added retroactively to a build already in someone's hands.

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

### Make the invariant structural when a centralised check can still be bypassed

Centralising a check behind one guard removes the copies, but calling it is still a convention — a
new call site that forgets it compiles and ships. When the cost of missing it is real, encode
"requests of this kind must go through that path" in the type system, so the wrong route fails to
compile rather than failing silently at runtime.

The tell that a convention isn't enough: count the call sites the guard actually covers. A
`DataManager`-level eviction check sat at four result checks and looked complete, but two of the
seven authenticated endpoints returned the connector's result directly and were never checked.

```csharp
// Before: every authenticated call had to remember to check the result
var result = await ServiceConnector.GetOrderYearsAsync(credentials);   // check omitted — silent gap

// After: the marker interface routes it, and only that path can carry it
private async Task<ServiceResult<T>> SendAuthenticatedMessage<T>(string path, IAuthenticatedMessage message)
```

### Don't identify a specific failure by its HTTP status alone

A status code is a namespace shared with infrastructure you don't control: a proxy, WAF, CDN, or
rate limiter can return 401/403/503 without your service being involved. Keying behaviour on the
status means acting on failures you didn't produce. Carry an application-level reason in the
response body and key on that; anything unparseable — an HTML challenge page, a truncated body —
then degrades to "no reason given" instead of impersonating a real one.

This also beats picking a "more correct" status. Moving credential rejection from 403 to 401 is
semantically better, but only trades a collision with the edge for a collision with the host's own
auth failures.

### Collapse a signal that every in-flight request will independently discover

When a state change invalidates work already in flight, each outstanding request discovers it
separately and will raise its own notification — so a user-visible response fires once per request
rather than once per event. Collapse it at the point that owns the state, and re-arm the latch when
the state can recur, or the second occurrence is swallowed forever.

Guards checked before a request is issued do not help here: everything already in flight passed them.

```csharp
// Fires once per session, not once per rejected request
if (Interlocked.Exchange(ref _sessionEvictionHandled, 1) == 1) { return; }
// ...and SetAccount resets it to 0, so signing back in can be evicted again
```

### A caller-side guard only suppresses the effects the caller owns

When a shared layer produces effects of its own — an error banner, a spinner, a log line — before it
hands control to the caller's handler, a check inside that handler cannot undo them. Moving the guard
into the shared layer is then not just less duplication; it is the only placement that reaches every
effect. Count the effects, not the call sites, when deciding where the guard goes.

```tsx
// Before: each page dropped a superseded response inside its own success handler — but the shared
// executeAction had already raised the error banner for a superseded *failure*, and clobbered the
// state a newer successful call had just cleared. No amount of page-side guarding reaches that.
okHandler: data => {
  if (seq !== loadSeq.current) { return; }
  setUsers(data.Items);
}

// After: the shared layer knows the call was superseded, so it skips every effect including its own
void executeAction({ action: signal => user.Service.GetUsers(request, signal), supersedeKey: 'users', ... })
```

### Collapse near-identical functions into one core parameterized by what differs

When several functions share the same skeleton and vary only along an axis or two, extract the
skeleton into one core that takes the varying parts as parameters (usually delegates), and reduce
each function to a thin wrapper. Parallel copies force every change to be pasted into each one, and
drift the moment a paste is missed.

```csharp
// Before: six HandleRequest* methods each repeated the same exception→response catch chain;
// adding a JsonException case meant pasting it into all six.

// After: one core owns the chain; each variant passes only the two things that differed
private async Task<HttpResponseData> HandleRequest(
    HttpRequestData request,
    Func<Task<HttpResponseData>> respond,
    Func<HttpStatusCode, string, Task<HttpResponseData>> createErrorResponse
)
```

### Prefer empty string over null for optional text

When an optional text field has no meaningful distinction between "null" and "empty", use a
non-nullable string with `""` as "no value". Nullability infects every consumer (null guards in
queries, optional types in generated contracts, `?? ''` mapping in forms) while buying nothing.
(Settled on Beverage.SKU/SupplierID, matching the existing ImageUrl convention.)

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

### Vary the value, not the mechanism, when two states differ only in appearance

When one state already has a mechanism, a second state that differs from it only in a value should
reuse that mechanism with a different value — not arrive with a mechanism of its own. The trap is
reaching for a second mechanism precisely _because_ the first is "taken" by another state: both then
render at once, they drift apart, and a reader has to hold two things where one would do.

```css
/* Before: the failing edge owned the outline, so the neutral edge arrived as a box-shadow —
   and a failing box drew both, one just outside the other */
.node {
  box-shadow: 0 0 0 calc(1px * var(--inv)) rgba(255, 255, 255, 0.16);
}
.node.has-fail {
  outline: calc(2px * var(--inv)) solid var(--fail);
}

/* After: one outline, recoloured per state */
.node {
  outline: calc(1px * var(--inv)) solid rgba(255, 255, 255, 0.16);
}
.node.has-fail {
  outline: calc(2px * var(--inv)) solid var(--fail);
}
```

### Don't back off a poll the user is sitting and waiting on

Exponential backoff assumes nobody is watching. When the poll drives a screen the user is blocked
on, the interval is at its longest exactly when they come back and expect the result — a 2s base
growing ×1.5 to a 10s cap is already capped after ~26s on screen, inside the window a verification
email arrives in. Keep the interval flat and short, and where the platform offers a signal that the
user has returned (app resume), cut the current wait short and poll immediately.

The load this trades away is worth it when the operation is rare and blocking: a login happens once
per user, so the extra requests are negligible against seconds of visible dead time.

```csharp
// Before: interval peaks right when the user returns from their email app
await Task.Delay(delayMs + Random.Shared.Next(0, 500));
delayMs = Math.Min(PollMaxDelayMs, (int)(delayMs * 1.5));

// After: flat and short, and the resume signal ends the wait early
var wake = _resumeWake.Task;
_ = await Task.WhenAny(wake, Task.Delay(PollIntervalMs + Random.Shared.Next(0, PollJitterMs)));
```

### Skip the work rather than throttling it when the precondition is absent

The cheaper half of the same problem: a poll that can't succeed — offline, or backgrounded — should
not run at all. Gating on the precondition removes both the request and the exception telemetry it
would produce, without slowing the poll down for the user who is actually waiting.

### Identify a user in telemetry by a stable opaque ID, never by PII or anything derived from it

The email is personal data, so it can't go to an analytics backend raw — but hashing it doesn't
solve the problem it looks like it solves. The hash is only as stable as its input, and the input is
something the user can edit: change the email and every event from before the change is orphaned
from every event after it. Use the account's own ID, which is opaque, not personal data, and doesn't
move when the profile does.

### Track an anonymous ID and a signed-in ID separately, and don't put the second in the first's field

The account ID only exists after login, so a design with one identity field has nothing to say about
everyone who hasn't signed in yet — and a shared placeholder is worse than nothing, because it merges
every signed-out user into a single phantom. Carry two: a random per-install ID generated once and
persisted, which is what the anonymous field is for, and the account ID in whatever field the backend
reserves for an authenticated identity. The install ID spans login and logout, so a user's pre-login
activity stays joined to what they do afterwards.

Check which field is which before assuming. Application Insights documents `user_Id` as the anonymous
ID and derives its sampling score from it — so a value that isn't random-ish skews sampling — and
says outright that identifying a signed-in user there is a misuse of the field; `user_AuthenticatedId`
is the one for that.

```csharp
// Before: the account ID in the anonymous field, and nothing at all before login
client.Context.User.Id = accountID;

// After
client.Context.User.Id = installID; // random GUID, persisted, survives logout
client.Context.User.AuthenticatedUserId = accountID; // null when signed out
```

### Measure "how long was the user away" with the wall clock, not a monotonic counter

A monotonic counter is the reflex for elapsed time because it can't be moved by a clock change — but
outside Windows it stops while the device is in a low-power state, and a backgrounded mobile app with
a locked phone is exactly that. An overnight absence then reads as minutes, and every gate built on it
silently never fires. `Environment.TickCount64` says so in its own docs: "The elapsed time excludes
any non-awake time, such as time spent in sleep mode, hibernation mode, or other low-power states."
The clock-change objection costs nothing here — a skewed clock produces one wrong session boundary,
where the counter produces a permanently wrong answer. Keep the counter for short in-process waits
where the device is awake throughout, such as a polling deadline.

```csharp
// Before: never trips after the phone sleeps
if (Environment.TickCount64 - _backgroundedAtTicks >= SessionTimeout.TotalMilliseconds)

// After
if (DateTimeOffset.UtcNow - _backgroundedAt >= SessionTimeout)
```

### Put an expensive or irreversible action behind a deliberate second keystroke

A single key that spends minutes of machine time, or that can't be undone, will eventually be hit by
accident. A two-key chord costs the deliberate user almost nothing and makes the accident
impossible. (Re-running a UI-test journey moved from `r` to `r` followed by `r`/`a`/`f` for its
scope.)

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

### Name both sides of a contrast, so neither name is the default

When a second API arrives that differs from an existing one along one axis, rename the original to
name that axis too. Leaving the first as the bare name implies it is the general case and the newcomer
a variant, and a reader choosing between them has nothing to go on but a guess.

```ts
// Before: the suffix says "value" but not how it differs, so the pair reads as general + special case
useDebounce; // owns the state
useDebouncedValue; // mirrors a value owned elsewhere

// After: both names carry the axis — who owns the value
useDebouncedState;
useDebouncedValue;
```

### Name a lookup for what it returns, not for what a caller might check

A lookup called `Verify*` reads as though it enforces something, so a caller may use it as a guard
and forget the comparison it was supposed to make. Name it for the value it produces, and let a
separate `Assert*` do the enforcing.

```csharp
VerifyCredentials  ->  ResolveAccountID   // returns an account ID, or null
                       AssertCredentials  // throws when the caller doesn't own the account
```

### Name a default enum member for the condition it represents, not as a placeholder

`Unknown` and `None` are easy to reach for and usually mean "I haven't decided what this member is
for". Name the actual condition instead. If no single name fits every case that lands on the
default, that's a sign the enum itself is misnamed — widen or narrow the type until one does.

```csharp
// Before: Unknown meant "no specific code" on the server and "couldn't parse it" on the client
public enum ServiceErrorCode { Unknown = 0, InvalidCredentials = 1 }

// After: naming the enum for refusals makes one honest name cover every default case —
// an internal error, a malformed request, and a response that never came from the service
public enum ServiceRejectionReason { NotApplicable = 0, InvalidCredentials = 1 }
```

### Name a wire type for its role in the exchange

`Request` and `Response` say which direction a type travels and what it's for. `Message` says only
that it crosses the wire, which every one of them does.

```csharp
SetOfferQuantityMessage   ->  SetOfferQuantityRequest
CardSetupSessionResponse  ->  CreateCardSetupSessionResponse
```

### Name a collection for what its elements are, not for the field you happen to render

A name drawn from one projection of the elements stops being true the moment a second field is used,
and it misleads a reader into thinking the collection holds that field. Name it for the elements.

```csharp
// Before: holds PostalCodeEntry records, but the picker only rendered .Town
public IReadOnlyList<PostalCodeEntry> TownOptions { get; set; } = [];

// After: the label later became City + Town, and the name was already right
public IReadOnlyList<PostalCodeEntry> Matches { get; set; } = [];
```

### Name a holder for its role, not the type it carries

A service or property that carries a value is named for the role it plays — where the value comes
from, what it's for — even when the value's own type is named for something else. Renaming the
holder to echo the type throws away the distinction the role name was drawing.

```csharp
// The app has two settings sources; the accessor names which is which.
AppSettings      // local device preferences
ServiceSettings  // settings from the service — a property of type PublicSettings

// PublicSettings names the wire projection's visibility (public vs the full admin doc);
// the app-side accessor names provenance. Renaming it PublicSettings would lose the
// local-vs-remote contrast with AppSettings.
```

### Draw a naming axis from the codebase's own module vocabulary

When two things contrast (local vs remote, read vs write), name them with the words the codebase
already uses for that split, not a synonym. Here the solution is `Beerbox.App.*` and
`Beerbox.Service.*`, so the settings sources are `AppSettings` / `ServiceSettings` — "Service", not
"Backend", even though "backend" is the looser everyday word. (`BackendUrl` / `IBackendConfiguration`
kept "Backend" — they name the connection endpoint, a different concept, and weren't part of the
split.)

### Name a mirrored third-party identifier for what it identifies, in their namespace not yours

A field holding another system's id needs two things the obvious name usually drops: whose id it is,
and what it points at. Naming it after your own entity is the worse failure — it reads as a foreign
key into your data and will be used as one. Reach for the vendor's own current vocabulary, but not
their legacy label: a term they keep for backwards compatibility describes what their product was,
not what it now covers.

```csharp
// Before: reads as our product id; it is not, and a bulk rename duly turned it into ProductID
public record UntappdDetails(int BeverageID, ...)

// After: whose id, and what it points at. Not `BeerID` — Untappd still says "beer" in its URLs,
// but its catalogue now covers cider, mead and spirits.
public record UntappdDetails(int UntappdBeverageID, ...)
```

## Renaming

### Before renaming a type or member, find out whether its name is also data

A rename is only a refactor while the name stays inside the compiler. The moment a framework derives
a string from it, renaming is a behaviour change that compiles clean and fails at runtime. Grep for
the mechanisms before starting, not after: `nameof` used as a storage or partition key,
`[CallerMemberName]`, `typeof(T).Name` / `GetType().Name`, and `T.FullName` as a cache key.

The consequences aren't uniform, so triage rather than treating them alike — an orphaned cache
self-heals on the next fetch, a changed preferences key silently resets a user's settings, and a
changed partition key strands every existing row where nothing queries it.

```csharp
// Renaming this record moves its whole partition; the old documents stay behind, invisible
public abstract record CosmosDBDocument(...)
{
    public string PartitionKey => GetType().Name;
}
```

### A rename can change what a read-modify-write loop writes to

When the renamed name selects a destination, a loop that used to read and write the same place
becomes a copy between two places. Code written for the in-place version is then subtly wrong in
ways the diff doesn't show — most often a missing delete, because deleting the source would have
destroyed the very record it just wrote. Re-read every loop that both reads and writes the renamed
thing.

```csharp
// Before the rename this upserted a Beverage back into the Beverage partition — an in-place
// upgrade, where deleting the source would have been wrong. After it, the upsert lands in the
// Product partition and the source has to be deleted, or the next step re-reads and clobbers it.
_ = await Database.UpsertItemAsync(product);
await Database.DeleteItemAsync((string)old["id"]!, "Beverage");
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

### Ask whether a reader who never saw the diff would be confused

The rules in this section are easy to agree with and still walk past, because at write time the
thought is never "I am rationalizing" — it's "this is subtle, let me explain". They only fire if you
ask a question you can answer while writing: would someone opening this file cold, not knowing what
the code used to be, be confused without this? If the comment only lands for a reader comparing
before and after, it's written for the reviewer — commit message, not source.

The tell is where the comments sit. Clustering on the lines you expect to be _questioned_ rather
than the lines that are hard to _read_ means you're defending the change, not documenting the code.
Those two overlap; they are not the same set. Recognisable shapes, each a restatement of one of the
rules around it:

- `rather than`, `instead of`, `the old`, `used to` — a rejected alternative, or history
- a sentence that paraphrases the statement directly beneath it — restatement
- a comment describing another file's or layer's behaviour — that layer's contract, not this one's

Judge them as a set rather than one at a time. Each is individually defensible; that is how a diff
ends up carrying fourteen of them.

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

### Canonicalize a value in the type that owns it, not in a helper at call sites

A normalization helper only fixes the call sites someone remembered to apply it to; every comparison
site it misses is a silent mismatch. Putting the canonicalization on the owning property covers every
consumer at once. (Moving email trim+lowercase from an `AccountManager.NormalizeEmail` helper into
`AccountCredentials.EmailAddress` fixed two lookup paths the helper-based fix had missed.)

```csharp
// Before: applied in one manager method; other comparisons of EmailAddress stayed raw
public static string NormalizeEmail(string email) => email.Trim().ToLowerInvariant();

// After: every consumer reads the canonical form
public string EmailAddress => ID?.Trim().ToLowerInvariant()!;
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

### Don't add a member that only forwards to another member

A property that reads and writes another object's property, adding no validation, no invariant, and
no behaviour, is a second name for one thing. If it exists to give call sites a better name, put the
name on the field instead — one name, in one place. "It insulates callers from the backing store"
isn't a reason: the interface is already that seam.

```csharp
// Before: a protected member whose whole job is to rename the store's Token
protected string? ClientToken
{
    get => _tokenStore.Token;
    set => _tokenStore.Token = value;
}

// After: the field carries the name, and the 8 call sites read _clientTokenStore.Token
private readonly ISecureTokenStore _clientTokenStore = clientTokenStore;
```

### Don't mark a type for codegen export when nothing downstream consumes it

An opt-in export annotation is a claim that a consumer needs the type, so adding it by reflex emits
generated files nobody imports. It also pulls in every type the exported one references, so a single
unnecessary annotation can produce several dead artifacts. Check what the consumer actually imports;
in a codebase where the annotation is opt-in, the unannotated types are the convention, not an
oversight.

```csharp
// The webapp imports none of these, and annotating PostalCodeEntry also emitted a Prefecture.ts
// that nothing references.
[ExportTsInterface]
public record PostalCodeEntry(string PostalCode, Prefecture Prefecture, string City, string Town);
```

### Don't write migration code for a state the product has never been in

Back-compat paths — migrating old data, tolerating a superseded format — are only worth their weight
once installs exist that could be in that state. Before launch there are none, so the migration is
untestable code guarding an impossible case.

## Validation

### Reject a shape that has no correct use, rather than leaving it legal and documenting it

If a construct can be built but can never be the right thing to build, make construction throw and
put the fix in the message. Documentation is opt-in and a comment is easy to miss; a throw reaches
whoever writes the mistake, at the moment they write it.

```csharp
// A tree leaf with no steps of its own re-ran the path its siblings already covered, asserted
// nothing they didn't, and produced no screenshot — a whole test execution per fixture for nothing.
public sealed class Leaf(JourneyStep[] steps, [CallerMemberName] string name = "")
    : TreeNode(
        name,
        steps.Length > 0
            ? steps
            : throw new ArgumentException(
                $"Leaf '{name}' has no steps of its own, so it duplicates the path its siblings "
                    + "already run. Give it steps, or move its expectations into the shared "
                    + "branch step and remove the leaf.",
                nameof(steps)
            )
    );
```

### Fail where a bad value is produced, not where it is consumed

A step that writes an empty or invalid value onward keeps going, and the build or request dies later
inside whatever consumes it — so the error names the consumer, and the reader starts investigating the
wrong file. Guard at the point of production, where the cause is still in hand, and put the cause in
the message.

```xml
<!-- Stamping an empty versionCode succeeded here, then failed ~40s later in the Android resource
     compiler with three APT2140 errors all blaming AndroidManifest.xml -->
<Error
  Condition="'$(BuildVersion)' == ''"
  Text="Nerdbank.GitVersioning produced no version, so android:versionCode would be stamped empty.
        This is usually an in-progress merge or rebase — finish or abort it, then rebuild."
/>
```

### The server validates cross-record constraints; the form validates field content

A constraint that spans records — a reference must exist, a value must be unique among its peers —
can only be checked where the records live, so it belongs in the server's write path (and is
consistent with the existing reference-exists and no-overlap checks there). Whether a single field
is non-empty or well-formed depends only on itself, and stays in the form's validation. Don't read
the absence of server-side field checks as a reason to drop the cross-record ones.

### Don't require in the form a field the server treats as optional

Before adding a field to a form's validity check, confirm the write path actually demands it. An
optional field made required blocks legitimate records, and every server test stays green — the
server never rejected it in the first place, so nothing goes red. Pin the optionality with a test
asserting the form stays submittable while the field is empty; without one, nothing records that the
omission was deliberate, and the next pass that tightens validation takes it out silently.

```tsx
// SKU is optional — an empty one is exempt from the (SupplierID, SKU) uniqueness check — so it
// stays out of isFormValid, and this is the only thing saying so
it("leaves create enabled when the SKU is empty", async () => {
  await fillRequiredFields(user);

  expect(screen.getByRole("button", { name: "Create" })).toBeEnabled();
});
```

### Validate an untrusted path against the resolved full path, not its segments

When a filesystem path is built from untrusted input — URL segments, request fields — reject it by
resolving the whole candidate and checking containment (`Path.GetFullPath(candidate)` starts with
`Path.GetFullPath(root) + separator`), not by scanning the raw or unescaped segments for `..`. A
per-segment `..` check reads as sufficient but isn't: a percent-encoded slash (`..%2f..`) decodes
into a single segment _after_ the URL is split, and a leading `%2f` decodes to an absolute segment
that `Path.Combine` honours outright — both slip past a whole-segment comparison. Put the guard
where the path is finally resolved (the read and the delete), and keep it even for a localhost-only
tool: a page the user visits can drive it cross-origin.

## Tests

### Wait for readiness explicitly instead of letting the first assertion absorb startup

An assertion's timeout is sized for the thing it asserts, not for a cold start. When the first
assertion after a launch also has to cover process start, a launch that is merely slow reports as a
missing element, and the test looks flaky rather than slow. Make readiness part of starting the
subject — wait for it to be foregrounded and settled — so the assertion's budget is spent only on
the assertion.

### State the elapsed time when a test asserts that nothing happened

An assertion that something is still absent, or still on screen, is satisfied the instant it runs —
so a step that only asserts it pins the state as it already was, not that it persisted. When the
claim is a negative over time ("no poll fired", "no retry was issued"), make the wait an explicit
part of the test, sized against the interval it has to outlast. Reusing a no-op step and relying on
whatever incidental delay the harness contributes can pass for a while, but the margin is invisible
and shrinks silently when the harness gets faster.

```csharp
// Before: expectations run immediately; both are already true, so the step proves nothing
Step(None(), [Found(Id.WaitForVerification.Cancel), NotFound(Id.Offer.BeverageName)])

// After: the 2s is the claim — longer than the 1s poll interval it must outlast
Step(Wait(2000), [Found(Id.WaitForVerification.Cancel), NotFound(Id.Offer.BeverageName)])
```

### A concurrency test against a zero-latency fake isn't testing concurrency

A fake that returns immediately runs "concurrent" calls one at a time, so each completes before the
next begins and any short-circuit guard hides the overlap the test exists to cover. The test then
passes with the fix removed. Induce latency in the fake so the calls are genuinely in flight
together, then confirm it goes red without the fix.

```csharp
// Without this the three calls complete serially and evictedCount is 1 either way
manager.MockServiceConnector.ArtificialLatencyMillis = 50;
Task.WaitAll(a, b, c);   // now fails with "found 3" when the latch is removed
```

### A test asserts its marginal behavior, not a copy of another test's expectations

When a second test exists only to pin one additional behavior (an error propagates, a status passes
through), match the collaborator loosely and assert just that behavior. Duplicating another test's
exact-match setup adds a second copy that must be edited in lockstep while verifying nothing the
first test doesn't already cover — and a test whose only marginal assertion is default behavior
(e.g. an exception propagating through code with no try/catch) may not be worth keeping at all.

### A snapshot suite's filenames passing says nothing about its contents

Golden-file tooling usually offers a cheap structural check — every stored file matches an expected
path — and it is easy to read a clean result as "the baselines are fine". It only proves the names
line up. Any change to what is rendered invalidates the pixels while leaving every path intact, so
the check stays green and the suite fails on the next real run.

The trap is that the rendering change need not look like one. A rename that touches only identifiers
is safe; the same commit editing one user-visible string is not, and nothing in a compiler, a unit
test, or the path check distinguishes them. Open a baseline and look at it.

## Evidence

### Prefer established, documented practice over an invented pattern

When proposing something novel, check whether it's actually an established practice, and say plainly
when it isn't. A pattern transferred from another domain is a reasoned guess, not received wisdom —
present it as one. An unusual pattern also costs future readers, who won't recognise it and have no
community experience to draw on.

### Prove a guard can fail

A test that cannot fail is not protection. Deliberately break the thing it guards and confirm it
goes red before claiming it works.

### Run a format parser over the real corpus, not just hand-written fixtures

Fixtures encode the cases you already thought of, so they confirm the parser rather than test it. The
real file carries the distribution — the placeholder rows, the near-duplicates, the records that
break an assumption you didn't know you'd made — and finding those costs one throwaway run. Turn what
it finds into fixtures, so the regression is pinned without the corpus.

```
Downloading Japan Post's 124,513-row file and running the importer over it found two defects
that every hand-written fixture had passed: rows that collapsed to duplicates once a
parenthesised note was stripped, and 134 codes spanning more than one city, which made a
picker label render blank.
```

### Capture failure evidence before running the diagnostics that explain it

Evidence is only worth what it depicted at the moment of failure. Querying process state, pulling a
device log, or any other diagnostic buys seconds — long enough for a screen that was merely slow to
finish rendering, so the screenshot then shows a healthy state that contradicts the failure it is
filed under, and the next reader chases a phantom. Grab the evidence first, then diagnose.

## Reading the situation

### Asymmetry is a smell

When similar things are treated differently for no stated reason, the difference is usually neglect
rather than intent, and is worth pulling on. (Two folders declared and a third not; ten copy rules
for seven files.)
