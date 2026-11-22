# `.docs/` — private workspace documentation

The user's private documentation for this workspace: one folder per repository{{SHARED_LINE}}. The placement and
reference rules are in the `manage-docs` skill (`.claude/skills/manage-docs/SKILL.md`). This file describes the tree.

- **Private.** Never shared or pushed, and nothing inside a repository may link to it.
- **Its own local git repository.** Every change is committed right away, one commit per logical change
  (`docs(<area>): …`). There is no remote, and history is never rewritten.
- **Each repository's `docs/` belongs to the team** and is committed with that repo. Read it there; `.docs/` keeps
  no copies. Agents never edit an existing file in it. After a feature passes its final test, they add one new
  {{LANGUAGE}} feature document.

## Read first

| File | What it holds |
|---|---|
| `<repo>/agents/conventions.md` | Rules for agents working in that repo (created when the first rule is settled) |
| `shared/notes/product-decisions.md` | Product decisions that look like bugs but are deliberate (create when needed) |

## Layout

```
{{TREE}}
```

## Which folder does a document go in?

| Document | Location |
|---|---|
| Feature work (spec + tickets) | `<repo or shared>/issues/<feature>/spec.md` and `issues/NN-<slug>.md` |
| Design spec outside the ticket flow | `<repo>/specs/YYYY-MM-DD-<slug>-design.md` |
| Implementation plan | `<repo>/plans/YYYY-MM-DD-<slug>.md` |
| Anything touching two or more repos | the same under `shared/`, never duplicated into both repo folders |
| Architecture decision record | `<repo>/adr/NNNN-<slug>.md` |
| Reference documentation, findings, runbooks | `<repo>/notes/` or `shared/notes/` |
| Finished one-off material | `shared/archive/<original path>` |

## Conventions

- {{LANGUAGE}} for every file; the conversation stays in the user's language.
- A spec and its plan share the date and slug. Every plan opens with a `**Spec:**` line giving the full `.docs/…`
  path.
- Every `.docs/…` link resolves: rewrite references when moving or deleting a file.
- When a spec reverses an earlier one, add a supersession notice naming the reversed clauses; do not delete the old
  spec.
- Dates are the date of writing.
- Add new top-level folders only together with an update to this file and the `manage-docs` skill.
