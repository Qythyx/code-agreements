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
field. Make the tolerance opt-in per enum, so one that only travels toward an always-current peer
stays strict. The fallback has to ship in the client's _first_ release; it can't be added
retroactively to a build already in someone's hands.

### Centralise a repeated correctness check behind one named guard

Duplicating a check is worse than duplicating a fact: every hand-written copy is a fresh chance to
invert the comparison, forget the throw, or pick the wrong status code. Where the check enforces a
security invariant, a bad copy is a vulnerability rather than drift.

Calling the guard is still a convention, though, and a new call site that forgets it compiles and
ships. Where the cost of missing it is real, encode the routing in the type system so the wrong path
fails to compile. The tell that a convention isn't enough: count the call sites the guard actually
covers, not the ones you remember writing.

```csharp
// Before: written out at seven call sites
if (await ResolveAccountID(credentials) != credentials.AccountID)
{
    throw new DatabaseException(HttpStatusCode.Forbidden, "Invalid credentials for the requested account");
}

// After
await AssertCredentials(credentials);
```

### Don't identify a specific failure by its HTTP status alone

A status code is a namespace shared with infrastructure you don't control: a proxy, WAF, CDN, or
rate limiter can return 401/403/503 without your service being involved. Keying behaviour on the
status means acting on failures you didn't produce. Carry an application-level reason in the
response body and key on that; anything unparseable then degrades to "no reason given" instead of
impersonating a real one. Picking a "more correct" status is not the fix — it only trades a
collision with the edge for one with the host's own auth failures.

### A caller-side guard only suppresses the effects the caller owns

When a shared layer produces effects of its own — an error banner, a spinner, a log line — before it
hands control to the caller's handler, a check inside that handler cannot undo them. Moving the
guard into the shared layer is then not just less duplication; it is the only placement that reaches
every effect. Count the effects, not the call sites, when deciding where the guard goes.

### Collapse near-identical functions into one core parameterized by what differs

When several functions share the same skeleton and vary only along an axis or two, extract the
skeleton into one core that takes the varying parts as parameters (usually delegates), and reduce
each function to a thin wrapper. Parallel copies force every change to be pasted into each one, and
drift the moment a paste is missed.

### Prefer empty string over null for optional text

When an optional text field has no meaningful distinction between "null" and "empty", use a
non-nullable string with `""` as "no value". Nullability infects every consumer (null guards in
queries, optional types in generated contracts, `?? ''` mapping in forms) while buying nothing.
(Settled on Beverage.SKU/SupplierID, matching the existing ImageUrl convention.)

### Avoid variables and helpers that are used only once

Inline the expression unless naming it genuinely aids reading or improves the formatting. This
covers small helper functions too: a one-call-site helper whose "why" is already carried by a
comment at the call site is pure indirection.

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
  box-shadow: 0 0 0 1px rgba(255, 255, 255, 0.16);
}
.node.has-fail {
  outline: 2px solid var(--fail);
}

/* After: one outline, recoloured per state */
.node {
  outline: 1px solid rgba(255, 255, 255, 0.16);
}
.node.has-fail {
  outline: 2px solid var(--fail);
}
```

### Identify a user in telemetry by a stable opaque ID, never by PII or anything derived from it

Hashing the email doesn't solve the problem it looks like it solves: the hash is only as stable as
its input, and the user can edit that input, orphaning every event from before the change. Use the
account's own ID, which is opaque and doesn't move when the profile does.

### Refetch after a mutation when the server owns a derived value

Patching the changed row into local state is tempting because it is instant, but it only updates
what the client can recompute. Anything the server derived — a group's totals, a count, a rollup —
stays at its pre-mutation value, and the row and its header disagree on screen. Refetch instead, and
keep one source of truth.

### When fixing a defect inside an expression, change the wrong term and nothing else

Restructuring the expression around the fix makes the diff about your preferences rather than the
bug, and the reviewer then has to re-derive the behaviour change from a rewritten shape. It also
risks reverting the fix if they reject the restructuring — which is exactly what happened here twice
before the one-token change landed. A named intermediate that states a condition's intent survives
this even at a single use site; "used once" is not on its own a reason to inline it.

```csharp
// The whole fix: one comparison. wantPersonalRating stays, and the surrounding lines stay.
var showGlobalRating =
    _settings.UntappdSetting.HasFlag(UntappdSetting.GlobalRating)
    || (wantPersonalRating && personalRating == 0);
```

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
name that axis too. Leaving the first as the bare name implies it is the general case and the
newcomer a variant, and a reader choosing between them has nothing to go on but a guess.

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

The converse holds too: `Result` and `Response` on a type that never crosses the wire gets it filed
with the contracts and eventually annotated like one. Name an internal hand-off for what it is.

```csharp
SetOfferQuantityMessage   ->  SetOfferQuantityRequest
CardSetupSessionResponse  ->  CreateCardSetupSessionResponse

LoginResult(ClientUser User, string SessionToken)  ->  MintedSession(ClientUser User, string Token)
```

### Default an arbitrary string value to lowercase

Cookie names, header names, storage keys and other strings the language doesn't constrain default to
lowercase. Capitalise only where a language convention requires it, such as a C# type name.

```csharp
"BeerboxSession"  ->  "beerboxSession"
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

A property that carries a value is named for the role it plays — where the value comes from, what
it's for — even when the value's own type is named for something else. Draw the contrast from the
words the codebase already uses for that split, not a synonym: with `App.*` and `Service.*`
assemblies, the two settings sources are `AppSettings` and `ServiceSettings`, not `BackendSettings`.

```csharp
AppSettings      // local device preferences
ServiceSettings  // from the service — a property of type PublicSettings, and named for provenance
                 // rather than echoing the type, which would lose the contrast with AppSettings
```

### Name a mirrored third-party identifier for what it identifies, in their namespace not yours

A field holding another system's id needs two things the obvious name usually drops: whose id it is,
and what it points at. Naming it after your own entity is the worse failure — it reads as a foreign
key into your data and will be used as one. Reach for the vendor's current vocabulary, not the
legacy label they keep for backwards compatibility.

```csharp
// Before: reads as our product id; it is not, and a bulk rename duly turned it into ProductID
public record UntappdDetails(int BeverageID, ...)

// After: whose id, and what it points at
public record UntappdDetails(int UntappdBeverageID, ...)
```

### Name an outcome for what happened, not for what the caller will do about it

A result named after its consequence bakes one caller's reaction into a value other callers have to
reinterpret, and it reads as a command rather than a report. Name the condition; let the caller name
the response.

```csharp
// Before: the name is the caller's reaction, decided in the wrong place
private enum ReadOutcome { NothingRead, Written, LinkDropped }

// After: what the service actually said. Dropping the link is what one caller then does.
private enum ReadOutcome { NothingRead, Written, TokenRejected }
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
// The condition and the message are the whole story — the paragraph that stood above this
// explaining what a negative quantity would do downstream said nothing the line doesn't.
if (quantity < 0)
{
    throw new DatabaseException(HttpStatusCode.BadRequest, "Quantity cannot be negative");
}
```

### Ask whether a reader who never saw the diff would be confused

These rules are easy to agree with and still walk past, because at write time the thought is never
"I am rationalizing" — it's "this is subtle, let me explain". They only fire if you ask a question
you can answer while writing: would someone opening this file cold, not knowing what the code used
to be, be confused without this? If the comment only lands for a reader comparing before and after,
it's written for the reviewer — commit message, not source.

The tell is where the comments sit. Clustering on the lines you expect to be _questioned_ rather
than the lines that are hard to _read_ means you're defending the change, not documenting the code.
Recognisable shapes:

- `rather than`, `instead of`, `the old`, `used to` — a rejected alternative, or history
- a sentence that paraphrases the statement directly beneath it — restatement
- a comment describing another file's or layer's behaviour — that layer's contract, not this one's
- `irreversible`, `one-way`, `cannot be undone` — a warning aimed at whoever _approves_ the change,
  not at whoever reads the file later; by then the one-way step is long taken

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

// After — a cause that can actually happen
// ...falling back to the most recently added card if no default is set — possible when the card
// attached on Stripe's side but the handler that records the default never ran.
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
// @semantic-release/git runs from this cwd and builds its list with `git ls-files`...

// After — only the coupling this line can't show on its own
// Bump version.json (repo root) and stage it for the release commit below.
prepareCmd: 'node scripts/bump-version-json.mjs ${v} && git add ../../version.json',
```

### Qualify a term that collides with a more common developer meaning

A word that means something else in the reader's daily context sends them down the wrong path before
the sentence finishes. "Breakpoint" reads as the debugger's; "layout breakpoint" doesn't.

## Warnings

### Never silence a warning to make it go away

Fix the underlying issue. If it genuinely can't be fixed reasonably, ask before suppressing it —
don't decide unilaterally that a warning is acceptable.

## Automation

### Automation that fires on a version-control event may write derived artifacts, never tracked files

A hook attached to checkout, merge, or rebase runs at a moment the user is thinking about git, not
about your script. Rebuilding derived state there is helpful; modifying a tracked file is a change
they didn't ask for, arriving with no diff to review, and it leaves the working tree dirty as a side
effect of moving between branches.

```bash
# Before: resolves against the manifest and can rewrite the lockfile mid-checkout
pnpm install

# After: can only write node_modules, and installs exactly what CI installs
pnpm install --frozen-lockfile
```

### A gate that only runs in CI will first fail in CI

If a check is worth blocking a merge on, wire it into a command developers already run so it fails
on their machine first. A separate "run this before pushing" script relies on memory, which is the
thing that just failed. Prefer attaching it to an existing verb over inventing a new one, and leave
the fast iteration paths unattached so the inner loop stays quick.

```jsonc
// `pnpm test` now enforces exactly what CI enforces; test:watch deliberately doesn't,
// so a red lint can't block iterating on a failing test.
"pretest": "pnpm run check",
```

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

### Put mapping and canonicalization in the type that owns the value, not in a helper at call sites

Not in the caller, and not in a satellite helper. A helper only covers the call sites someone
remembered to apply it to, and every site it misses is a silent mismatch — where putting the
behaviour on the owning property covers every consumer at once.

```csharp
// Before: a satellite formatter, and a normalizer applied in one manager method
public static string? ForCard(RegisteredPaymentMethod pm) => ...;
public static string NormalizeEmail(string email) => email.Trim().ToLowerInvariant();

// After: the type that holds the data formats and canonicalizes itself
public string Describe() => ...;
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
a second chance to fail.

### Use the single-match operation when only one element can match

A loop that keeps scanning after it has found its target is handling a case the invariant forbids,
and the reverse iteration it then needs to delete safely sends the reader looking for the
multiple-match scenario. Find the one element and act on it.

```csharp
// Before: a reverse loop so RemoveAt stays safe mid-iteration, for a second match that cannot exist
for (var i = properties.Count - 1; i >= 0; i--) { if (...) { properties.RemoveAt(i); } }

// After
if (properties.FirstOrDefault(p => ...) is { } partitionKey) { _ = properties.Remove(partitionKey); }
```

### Don't defend against a failure the codebase ignores elsewhere

If nothing else re-creates a deleted database item, a special case for one resource type is
inconsistent, not careful. Consistency with the surrounding code is itself an argument.

### Prefer removing a hazard to documenting it

When a trap exists only because something is inconsistent, fix the inconsistency instead of writing
the warning. A documented hazard still catches people, and the document rots independently of the
thing it describes.

```
Four test projects, three AssemblyName patterns, two resolving differently in Debug and Release.

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
protected string? ClientToken { get => _tokenStore.Token; set => _tokenStore.Token = value; }

// After: the field carries the name, and the 8 call sites read _clientTokenStore.Token
private readonly ISecureTokenStore _clientTokenStore = clientTokenStore;
```

### Don't write migration code for a state the product has never been in

Back-compat paths — migrating old data, tolerating a superseded format — are only worth their weight
once installs exist that could be in that state. Before launch there are none, so the migration is
untestable code guarding an impossible case.

### Tolerate an absent value only where a record in that shape can exist

The same field can warrant different answers in different stores, and symmetry between two models is
not the argument — evidence that something can read back in that shape is. Nullability and a default
are separate decisions: the first says the value can be missing, the second says the whole key can
be. Ask which writers exist before granting either.

### Satisfying the type-checker must not cost a name or add a variable

When a value turns nullable, add the check where the value is used. Restructuring so flow analysis
can carry non-nullness to the use site — hoisting the object into a new local, replacing a
well-named flag with a nullable holder — buys a mechanical guarantee with the reader's
understanding, which is the wrong trade. The tempting restructure exists because a `bool` cannot
carry the non-null fact to a dereference further down; a check at the dereference costs less.

```csharp
// Rejected: renamed, and a third local, so the compiler could see through to the dereference
var globalRatingSource = untappd is not null && (...) ? untappd : null;
globalRatingSource is not null ? RenderStarRating(..., globalRatingSource.GlobalRating, ...) : null

// Accepted: same names, same shape, nullability handled where it arises
var showGlobalRating = setting.HasFlag(GlobalRating) || (untappd?.PersonalRating == 0 && ...);
untappd is not null && showGlobalRating ? RenderStarRating(..., untappd.GlobalRating, ...) : null
```

### Don't hand-roll a guarantee the platform already makes

Check the platform's documented behaviour before building mutual exclusion, retry, or overlap
detection. Beyond being redundant, a hand-rolled version is usually _worse_: the platform's is a
lease that expires, while a flag in your own store is a poison pill — a run killed mid-flight leaves
it set and silently blocks every later run.

```
Azure Functions timer triggers, verbatim: "only a single instance of a timer-triggered function is
run across all instances. It will not trigger again if there is an outstanding invocation still
running." Overlap detection in the manager would have duplicated that with a worse mechanism.
```

### Remove a limit whose value is a guess, when exceeding it fails loudly and capping it fails silently

A guessed cap cannot reliably prevent the failure it was added for: set too high it never binds, set
too low it throttles throughput with no signal. Prefer the loud failure — it names the problem and
suggests the fix — over a silent ceiling nobody will revisit. This only holds when overrunning is
safe: check that partial progress is durable and that the next run resumes where this one stopped.

## Validation

### Reject a shape that has no correct use, rather than leaving it legal and documenting it

If a construct can be built but can never be the right thing to build, make construction throw and
put the fix in the message. Documentation is opt-in and a comment is easy to miss; a throw reaches
whoever writes the mistake, at the moment they write it.

```csharp
// The message carries the fix, so it reaches whoever writes the mistake
steps.Length > 0
    ? steps
    : throw new ArgumentException(
        $"Leaf '{name}' has no steps of its own, so it duplicates the path its siblings already "
            + "run. Give it steps, or move its expectations into the shared branch step.",
        nameof(steps)
    )
```

### Enforce a server-owned invariant at the server, not in the client's type system

When a value is the server's to decide, a client type shaped to avoid sending a wrong one is a
workaround: every other client has to repeat it, and a hand-rolled request bypasses it entirely.
Move the check into the write path and let the client's types stay plain — the guard usually costs
less than the workaround it replaces.

```ts
// Before: a webapp-only type existed solely to keep a blank id out of the payload
type SettingsDocument = Pick<API.Settings, "_etag"> & SettingsFormData;

// After: state is the generated type again, because the service now owns the id
const [settings, setSettings] = useState<API.Settings | undefined>(undefined);
```

### Fill in what the caller could not know; reject what it got wrong

Absent and incorrect are different errors and deserve different answers. A caller that omits a value
it has no way to know — the id of a singleton document it has never been told — should have it
filled in. A caller that supplies a _different_ value is mistaken or probing, and quietly rewriting
that hides it. Split the guard so the loud case stays loud.

```csharp
// An empty id is the blank admin form creating the document; any other id names something else
if (!string.IsNullOrEmpty(settings.ID) && settings.ID != Settings.SettingsID)
{
    throw new ArgumentException($"The settings document id must be '{Settings.SettingsID}'.", nameof(settings));
}
settings = settings with { ID = Settings.SettingsID };
```

### Fail where a bad value is produced, not where it is consumed

A step that writes an empty or invalid value onward keeps going, and the build or request dies later
inside whatever consumes it — so the error names the consumer, and the reader starts investigating
the wrong file. Guard at the point of production, where the cause is still in hand, and put the
cause in the message.

```xml
<!-- Stamping an empty versionCode succeeded here, then failed ~40s later in the Android resource
     compiler with three errors all blaming AndroidManifest.xml -->
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

### Validate an untrusted path against the resolved full path, not its segments

When a filesystem path is built from untrusted input — URL segments, request fields — reject it by
resolving the whole candidate and checking containment (`Path.GetFullPath(candidate)` starts with
`Path.GetFullPath(root) + separator`), not by scanning the raw or unescaped segments for `..`. A
per-segment `..` check reads as sufficient but isn't: a percent-encoded slash (`..%2f..`) decodes
into a single segment _after_ the URL is split, and a leading `%2f` decodes to an absolute segment
that `Path.Combine` honours outright. Put the guard where the path is finally resolved, and keep it
even for a localhost-only tool: a page the user visits can drive it cross-origin.

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
passes with the fix removed. Require the overlap rather than sampling it — have each caller wait
until N are in flight together, which is deterministic and fails rather than hangs when the code
turns out to be sequential. Sampling a high-water mark instead is a non-atomic read-modify-write, so
the thread that observed the peak can be overwritten by one that observed less.

```csharp
// Completes only on genuine overlap; sequential code leaves each caller waiting out the timeout
if (Interlocked.Increment(ref inFlight) == count) { _ = allInFlight.TrySetResult(); }
_ = await Task.WhenAny(allInFlight.Task, Task.Delay(TimeSpan.FromSeconds(2)));
```

### A test asserts its marginal behavior, not a copy of another test's expectations

When a second test exists only to pin one additional behavior (an error propagates, a status passes
through), match the collaborator loosely and assert just that behavior. Duplicating another test's
exact-match setup adds a second copy that must be edited in lockstep while verifying nothing the
first test doesn't already cover — and a test whose only marginal assertion is default behavior
(e.g. an exception propagating through code with no try/catch) may not be worth keeping at all.

### A structural check over a snapshot suite says nothing about its contents

Golden-file tooling offers a cheap check that every stored file matches an expected path, and a
clean result reads as "the baselines are fine". It only proves the names line up — any change to
what is rendered invalidates the pixels while leaving every path intact. Open a baseline and look at
it.

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

Fixtures encode the cases you already thought of, so they confirm the parser rather than test it.
The real file carries the distribution — the placeholder rows, the near-duplicates, the records that
break an assumption you didn't know you'd made — and finding those costs one throwaway run. Turn
what it finds into fixtures, so the regression is pinned without the corpus.

```
Running the importer over Japan Post's real 124,513-row file found two defects every hand-written
fixture had passed: rows that collapsed to duplicates once a parenthesised note was stripped, and
codes spanning more than one city, which made a picker label render blank.
```

### A tool accepting a flag is not evidence that the flag does anything

Many tools ignore unrecognised options rather than rejecting them, so a clean exit says only that
nothing crashed. Confirm the option exists in the documentation, or run the control: pass a
deliberately nonsensical name and see whether the tool objects. If it doesn't, acceptance proved
nothing and the flag has to be verified by its effect instead.

```
$ pnpm install --config.thisIsNotARealSetting=false --frozen-lockfile
Already up to date
Done in 159ms                       # exit 0 — pnpm ignores unknown config keys entirely
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
