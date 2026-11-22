---
name: docs-two-homes
description: "Two doc homes — private root .docs/ (default; own local git repo, auto-commit every change, never linked from a repo) and each repo's shared docs/ (never edit existing files; one new {{LANGUAGE}} feature doc after the final test). Load the manage-docs skill before any doc work."
metadata:
  type: feedback
---

Documentation in this workspace has two homes. The full rule is in the project skill `manage-docs`
(`.claude/skills/manage-docs/SKILL.md`). Load it before writing, moving or referencing any document.

- **Private `.docs/` at the workspace root is the default.** Specs, tickets, plans, ADRs, notes, reviews and
  runbooks go there. It is its own local git repo: commit every change right away with `docs(<area>): …`, never
  push, never rewrite history.
- **Each repo's `docs/` belongs to colleagues.** Never edit, move or delete an existing file there. The only write
  allowed is one **new** {{LANGUAGE}} feature doc per touched repo after the feature passes its final test. When it
  supersedes an older statement, name that document and quote it in a "Relation to existing docs" section.
- **One-way references.** No `.docs/` path, ticket number or private ADR number may appear in any repo file (code
  comment, test, commit message, PR text, README, `docs/`). Write the reasoning inline instead. Before handing work
  back, grep each touched repo for `.docs/`.
- **Agents that do not load skills** (other CLIs, worker panes) must get this rule in their brief: "no `.docs/`
  paths or ticket numbers in repo files; never edit existing `docs/` files".
- **Language:** files on disk in {{LANGUAGE}}; chat in the user's language.

**Why:** the user keeps personal working documents separate from the team's shared documentation. Colleagues own the
repo docs and their git history, and other developers do not have `.docs/`, so a reference to it withholds the
reason.

**How to apply:** default every new document to `.docs/`. Touch a repo's `docs/` only to add the finished-feature
doc. Commit `.docs/` after each change.
