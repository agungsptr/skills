## Documentation

Documentation has two homes:

- **Root `.docs/` — private, the default.** It has one folder per repository (plus `.docs/shared/` for work across
  repos), each with `agents/` · `issues/` · `specs/` · `plans/` · `adr/` · `notes/`. Specs, tickets, plans,
  reviews, runbooks and notes go here. `.docs/` is the user's own local git repo: it is never shared or pushed, and
  every change is committed right away (`docs(<area>): …`). Nothing in a repository may link to it.
- **Each repo's `docs/` — shared with the team, committed.** Never edit, move or delete an existing file there.
  After a feature passes its final test, add **one new** {{LANGUAGE}} document explaining the finished feature, as
  context for other programmers and their agents. When it changes or retires something an older document says, name
  that document and the statement in the new one.

Load the `manage-docs` skill (`.claude/skills/manage-docs/SKILL.md`) before writing, moving or referencing any
document. `.docs/README.md` describes the tree.
