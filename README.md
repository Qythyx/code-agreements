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

## What doesn't belong here

- **Agent-behavior guidance** — how Claude should work, when to ask before editing, what to put in a
  commit message. That goes in `~/.claude/CLAUDE.md`. These files are about the code.
- **Facts about a particular session** — "the sproc wasn't registered in staging" is history. That
  belongs in the project's session summary.
- **One-offs** — if it can't be stated in a way that applies to code neither of us has seen, it
  isn't a rule.

## Maintenance

Maintained by the `code-agreements` skill (`~/.claude/skills/code-agreements/SKILL.md`), which adds
entries as guidance settles during a session and reconciles them at the end. Editing by hand is fine
— it's a normal repo.

Format: `## Topic` sections, `### The rule as a sentence` per entry, reasoning below it, and a real
code example where one clarifies. Formatted with `npx prettier --write .`.
