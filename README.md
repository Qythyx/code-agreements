# Code Agreements

How I want code written, across all of my projects.

## What to read

Always read `general.md`. Then read the file for **every** language the change touches. A change
spanning a C# backend and a TypeScript front end needs `csharp.md` and `typescript-react.md` both.

| File                  | Holds                                                    |
| --------------------- | -------------------------------------------------------- |
| `general.md`          | Language-agnostic design and process guidance            |
| `csharp.md`           | C# / .NET                                                |
| `typescript-react.md` | TypeScript, React, and the web front end                 |
| `observations.md`     | The skill's log of corrections. Don't read before coding |

Where a project's own CLAUDE.md says something different, the project wins.

## How a rule gets here

A rule is added or changed only after I have approved its wording.

The `code-agreements` skill (`~/.claude/skills/code-agreements/SKILL.md`) logs my corrections to
`observations.md` without asking. When it is run explicitly — by name, from `session-summary`, or
from `/ship` — it reviews every open observation and proposes a rule only when the same kind of
correction has happened twice in different code, or when I state a standing preference ("always",
"never"). Each proposal shows the whole entry as it would appear. Editing these files by hand is
fine too; it's a normal repo.

The skill also audits the rules themselves when I ask, and offers to when a file nears its line
budget or the last audit is more than six months old.

Last audit: 2026-09-21

## What belongs here

A rule is a decision that comes up routinely, in any project, and that a competent developer
wouldn't make my way without being told.

These don't belong:

- **How Claude should work** — when to ask, what to verify, what goes in a commit message. That goes
  in `~/.claude/CLAUDE.md`.
- **One project's mechanics or conventions** — how its build is wired, how its serializer is
  configured, its naming scheme. That goes in the project's own CLAUDE.md.
- **Something found out while debugging.** The docs, the compiler, or the failure itself will teach
  it again.
- **Anything the compiler, analyzer, or linter already reports.** A rule adds nothing to a warning.
- **A special case of a rule already here.** Reword the general rule instead.

## Format

`## Topic` sections, one `### Heading` per rule.

- The heading is an instruction someone could follow without reading further. A default with
  exceptions starts with "Prefer", and the body says when the exception applies.
- The body is at most three plain sentences.
- Each sentence starts with the verb and tells the reader what to do, with the condition after it:
  "Inline a helper when it is only used once", not "A helper with a single use goes inline". Write
  full sentences, not fragments.
- The rule names a concrete action: "Put a feature's model and service in one folder", not "Group
  the related model and service together".
- A `When:` line appears only if it isn't obvious when the rule applies.
- An example appears only if the rule is ambiguous without one.

Route a rule by where it **applies**, not where it came up: a design principle that surfaced in C#
still belongs in `general.md`.

Line budgets: `general.md` 300, `csharp.md` 200, `typescript-react.md` 100. Going over means cutting
something. If you feel nothing should be cut then discuss it with the user. Formatted with
`npx prettier --write .`.
