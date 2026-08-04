# Code Agreements

How I want code written, across all of my projects.

## What this is

These files are read before writing code, so that guidance given once doesn't have to be given
again. Some of it was set down up front; most of it accumulates as things get settled during
development sessions — usually when I correct, simplify, or rename something.

Each entry records what was actually settled, with the reasoning attached, so a future session can
read it and skip the argument. It is not a style guide written from first principles.

## What to read

Always read `general.md`. It applies to every language, and it holds the design, comment, and process
guidance that is easiest to assume you already know.

Then read the file for **every** language the change touches — plural, not one. A change spanning a
C# backend and a TypeScript front end needs `csharp.md` and `typescript-react.md` both.

## Layout

| File                  | Holds                                         |
| --------------------- | --------------------------------------------- |
| `general.md`          | Language-agnostic design and process guidance |
| `csharp.md`           | C# / .NET                                     |
| `typescript-react.md` | TypeScript, React, and the web front end      |

Route an entry by where it **applies**, not where it came up: a design principle that surfaced in C#
still belongs in `general.md`.

## The bar for a new entry

These files are read in full before every coding task, so each rule taxes every future session
whether or not it applies. A new entry has to be **unguessable** (a competent implementer wouldn't
arrive at it unaided) and **recurrent** (it fires often, not just conceivably). Most candidates fail
one or the other. One entry is a normal yield from a session; five means a session's narrative is
being converted into rule-shaped prose.

The entries that earn their keep are mostly preferences — arbitrary, unguessable, two lines. Things
I _discovered_ while debugging generally don't: a future session reads it in the docs, or the
compiler says so, or the failure teaches it again in ten minutes.

## What doesn't belong here

- **Agent-behavior guidance** — how Claude should work, when to ask before editing, what to put in a
  commit message. That goes in `~/.claude/CLAUDE.md`. These files are about the code.
- **Facts about a particular session** — "the sproc wasn't registered in staging" is history. That
  belongs in the project's session summary.
- **War stories** — a rule whose first clause names a _situation_ ("A migration must not…", "Set
  every response header…") rather than a decision routinely faced. The wording generalizes; the
  trigger doesn't.
- **Special cases of a rule already here** — sharpen the general one instead of adding a second
  entry beside it.

## Maintenance

Maintained by the `code-agreements` skill (`~/.claude/skills/code-agreements/SKILL.md`), which adds
entries as guidance settles during a session and reconciles them at the end. Every run also prunes —
these files should get sharper over time, not longer. Editing by hand is fine; it's a normal repo.

Format: `## Topic` sections, `### The rule as a sentence` per entry, two to four lines of reasoning
below it, and at most one short code example where it clarifies. If an entry won't fit that, it's
carrying a situation with it and probably shouldn't be here. Formatted with `npx prettier --write .`.
