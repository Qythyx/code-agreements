# Code Agreements — General

Language-agnostic guidance that applies to every project. See [`README.md`](README.md) for how these
files are organised and maintained.

## Design

### Favour readability over performance

Write for the person reading the code next. Use a faster but harder-to-follow version only when a
measurement shows it is needed.

### Collapse near-identical functions into one

Make one function that takes the differing part as a parameter when two functions share the same
skeleton. Do this as soon as there are two copies; don't wait for a third. Do the same for two
endpoints that differ only by a verb.

```text
MarkAccountForDeletion(id) + ClearAccountDeletionMark(id)  ->  SetAccountDeletionMark(id, bool marked)
```

### Prefer inlining anything that is used only once

Inline a variable, helper, property, class, or UI component when it is only used once. Keep it
separate only when doing so makes the code easier to read; either having the separate name provides
clarity, or the inlined code is long enough that it would be disruptive inlined. Don't add a
property or method whose only job is to call another member; call that member directly.

### Reuse common UX components rather than styling each page

Identify the basic components and give them consistent styles. Define a unique style only when the
UX is genuinely unique.

### Don't store in component state what can be computed

Use state only for values that change through interaction. Compute a value at render time when it is
set once at mount, or derived from props or other state.

### Reload from the server after a save when the server calculates part of the data

Reload instead of patching the saved row into local state when the server derives totals, counts, or
rollups. Patching leaves those derived values stale, so the row and its summary disagree on screen.

## Naming

### Name a thing for what it is, not for what one caller does with it

Name what the thing is, does, returns, or identifies. Don't take the name from one caller's use,
because it stops being true when a second caller arrives. Name a delegate parameter for what it
does: `JsonProvider`, not `Json`.

```text
VerifyCredentials             ->  ResolveAccountID   // it returns an ID; it doesn't throw
enum ReadOutcome.LinkDropped  ->  TokenRejected      // what happened, not the caller's reaction
PublicSettings Settings       ->  ServiceSettings    // its role, not its type
```

### Choose a method's verb to say what the method actually does

Include what the methods it calls do. Don't write "Check" when the method also makes changes, "Find"
when it doesn't search, or "Process" or "Run" when a specific verb exists.

### Use words the reader already knows

Do this in names, comments, and docs. Don't invent a term ("closing pass", "slot") — say what the
thing does, even if that is longer. Qualify a word that has multiple potential meanings: "layout
breakpoint", not "breakpoint".

### Follow the naming conventions already established in the project

## Style

### Always use curly braces for code blocks

Use them even for single-line statements. The exception is a short inline lambda in React:
`onMouseDown={e => e.stopPropagation()}`.

### Prefer nested conditions over early returns, unless the nesting gets too deep

One exception is argument checks at the top of the method, for those an early return is fine.

## Comments

### Only comment in-code what the code can't say

Add documentation for public classes and methods. Add in-code comments only when the code is
genuinely tricky or unintuitive, or when the codebase requires documentation. Don't restate what the
next line does. Test each comment by asking what breaks if the reader ignores it; if nothing breaks,
it belongs in the commit message. Don't justify code with a scenario that can't happen in this
system.

## Documentation

### Draw diagrams in Markdown with Mermaid, not ASCII art

Use Mermaid because it renders on GitHub and survives edits. Keep directory and namespace trees as
text.

## Warnings

### Never silence a warning to make it go away

Fix the underlying issue. If it genuinely can't be fixed reasonably, ask before suppressing it.

## Encapsulation

### Ask a provider for what you want, not for how it is stored

Get a thing from the class that exists to provide it. Don't let a caller depend on that class's
folders, file names, or storage layout.

### Prefer a closed, typed API over an open-ended one

Make a public method take a value from a known set, such as an enum, not an arbitrary string. Pass a
method the value it reads, not a larger object that happens to carry it.

### Pass values that must agree as one type

Wrap two parameters in one type when they only make sense together, so a caller can't supply one
without the other.

```csharp
SendEmailAsync(to, subject, html, inlineImages)  ->  SendEmailAsync(to, subject, new EmailBody(html, inlineImages))
```

### Put formatting and normalising on the type that owns the value

Put trimming, lower-casing, and display formatting on the type that holds the data. Don't put them
in a helper that each caller has to remember to call. One exception to this is when the formatting
should be controlled by the frontend that displays it.

## Simplicity

### Don't add defensive code for a failure the rest of the codebase doesn't defend against

Check how the surrounding code treats the same failure before adding a catch, guard, or special
case. Leave it out when the failure fixes itself or another layer already handles it. If you feel it
is a genuine gap that should be closed ask before adding it.

### Prefer removing a hazard to documenting it

Change something that is easy to get wrong so that it can't be got wrong, instead of writing a
warning. Make building a construct fail when it can be built but is never correct, and put the fix
in the error message.

### Prefer one general rule over several specific ones

Avoid a rule per instance, because it becomes a list someone has to maintain. For example, write
`<EmbeddedResource Include="Resources/*/*" />`, not one entry per file.

### Don't write code for a state the product has never been in

Don't add migration, backwards-compatibility, or "not yet configured" handling until something can
actually be in that state; before launch there is no old data. Make a field nullable or defaulted
only if a record without it can really exist.

### Check what the platform provides before building it yourself

Check whether the platform or framework already provides locking, retry, overlap detection, or a
test fake (for example `FakeTimeProvider`) before building one.

### Don't add a limit whose value is a guess

Take a maximum length, size, or count from something real: what the client enforces, what real data
measures, what another path already bounds. Leave the limit out if nothing gives a number.

## Validation

### Enforce rules about the data on the server

Enforce rules that span records (a reference must exist, a value must be unique), and values the
server decides, in the server's write path. Check in the form only that each field is filled in and
well-formed. Duplicating the business logic checking on the frontend opens the possibility the
frontend and backend will diverge.

### Check a value where it is produced

Guard a value where it is created, and name the cause in the message. A bad value that travels
onward fails later, inside whatever consumes it, and the error points at the wrong place.

### Report failure when a check could not run

Fail the check when it can't get what it needs (a permission, a binary, a fetch that returned
nothing). Don't report "nothing wrong", and don't quietly check a smaller part.

## Tests

### Don't read the real clock in tests

Pass the time in. Give the test base one named, fixed instant (`MondayBeforeTheClose`) and express
other times relative to it.

### Never put real personal data in a fixture

Use values that are obviously fake: an invalid postal code, a `テスト` prefix.

### Assert only the one thing a test exists for

Match everything else loosely when a second test pins one extra behaviour, and assert just that
behaviour. Don't repeat another test's assertions.

### Delete a branch that only impossible input can reach

Delete the branch and its test when the only way to test it is input that production can't produce.

### Prove a test can fail

Break the code the test guards and confirm the test goes red. Two common tests that cannot fail: one
asserting a value equal to the type's default (`0`, `null`, `false`, empty), and a boundary test
whose input is not exactly on the boundary. This can also be achieved by writing the test first to
prove the bug you then fix.

## Evidence

### Prefer established, documented practice over an invented pattern

Check whether something novel is an established practice before proposing it, and say plainly when
it isn't. An unusual pattern costs future readers, who won't recognise it.

## Reading the situation

### Investigate when similar things are handled differently for no stated reason

Expect the difference to be an oversight. Example: two folders declared in a project file and a
third not.

### Don't leave one new file as the only one that follows a rule

Fix the whole set as part of the change when the existing files around the new one all break a rule
and the fix is small; ask first if it is large. Match the neighbours in the new file until the set
is fixed. If the change seems to disagree with the neighbours then ask which is correct.
