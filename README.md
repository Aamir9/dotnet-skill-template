# dotnet-skill-template

A starter `SKILL.md` for .NET projects using AI coding agents (Claude Code,
GitHub Copilot, Cursor, Codex, etc.).

Microsoft's [dotnet/skills](https://github.com/dotnet/skills) repo teaches
your agent general .NET best practices. It has no idea about **your**
project — your folder structure, whether your handlers return `Result<T>`,
or how your team implements CQRS. This template fills that gap.

Drop it in your project root, edit it to match your team's real decisions,
and your agent stops generating generic code and starts generating code
that looks like your codebase wrote it.

## What's inside

[`SKILL.md`](./SKILL.md) covers:

- **Architecture** — layering rules and dependency direction
- **CQRS handler pattern** — Command/Query + Handler + Validator, with a
  working code example
- **Result pattern** — no exceptions for expected/business failures
- **Naming conventions** — commands, queries, DTOs, interfaces, EF entities
- **EF Core rules** — avoiding N+1 queries, tracking behavior
- **Testing conventions** — real DB for integration tests, not mocks
- **What not to do** — explicit anti-patterns to keep the agent from
  over-engineering

## Quick start

1. Copy `SKILL.md` into the root of your .NET project (or your agent's
   equivalent skills folder).
2. Edit every section to reflect what your project *actually* does —
   this file is only useful if it's true. Delete anything that doesn't
   apply, and add the conventions that are missing.
3. Point your AI coding agent at it (most agents auto-load a `SKILL.md`
   / `CLAUDE.md` at the project root; check your tool's docs if not).

## A note on using AI skills

More skills loaded ≠ better results. Every skill an agent loads adds
tokens to every request and more surface area for it to misapply a rule
that doesn't fit the situation. Load only what matches your actual
workflow — a handful of precise, project-specific rules beats a hundred
generic ones.

## Related

- [Microsoft's official dotnet/skills](https://github.com/dotnet/skills) —
  general .NET best practices (EF Core, testing, ASP.NET Core, upgrades)
- This template — your project's specific conventions on top of that

## License

MIT — copy it, edit it, ship it.
