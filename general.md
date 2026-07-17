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

### Avoid variables that are used only once

Inline the expression unless naming it genuinely aids reading or improves the formatting.

### Reuse common UX components rather than styling each page

Identify the fundamental components and give them consistent styles. Define a unique style only when
the UX is genuinely unique — otherwise every page becomes its own small design system.

### Avoid hardcoded values, especially duplicated ones

Two literals that are the same value are one constant waiting to be extracted. Define the constant
and use it.

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

### Don't comment on what used to be there

A comment explaining what was removed, what this replaces, or what is no longer needed is written for
the reviewer, not the next reader. It's noise the moment the change merges.

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
