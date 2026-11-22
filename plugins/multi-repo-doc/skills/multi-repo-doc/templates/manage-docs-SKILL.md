---
name: manage-docs
description: Rules for where documentation goes in this workspace — the private root `.docs/` (default; specs, tickets, plans, ADRs, notes; auto-committed to its own local git repo) versus each repository's shared `docs/` (never edit existing files; one new {{LANGUAGE}} feature document after a feature passes its final test). Load before writing, moving, or referencing any document, and before writing a code comment that wants to cite a spec or decision.
---

# Managing docs: private `.docs/` vs shared `<repo>/docs/`

Workspace repos: {{REPOS}}. Every rule below applies to all of them. When a repo joins the workspace, add it here, to
the instruction block in `CLAUDE.md` and to `.docs/README.md`. Documentation has exactly two homes, with opposite
audiences.

| | `.docs/` (workspace root) | `<repo>/docs/` |
|---|---|---|
| Audience | the user and their agents only | the whole team and their agents |
| Committed / shared | **never** (own local git repo, no remote) | yes, committed with the repo |
| Holds | tickets, specs, plans, ADRs, notes, reviews, reports, runbooks | one finished-feature doc per feature |
| Language | {{LANGUAGE}} (chat stays in the user's language) | {{LANGUAGE}} |
| Edit existing files | yes | **never**; only add a new file |

`.docs/README.md` describes the tree; this skill is the rule.

## 1. Where a private document goes

| Document | Path |
|---|---|
| Feature work (spec + tickets) | `.docs/<repo or shared>/issues/<feature>/spec.md` and `issues/NN-<slug>.md` |
| Design spec outside the ticket flow | `.docs/<repo>/specs/YYYY-MM-DD-<slug>-design.md` |
| Implementation plan | `.docs/<repo>/plans/YYYY-MM-DD-<slug>.md` (first line: `**Spec:** <full .docs path>`) |
| Work touching two or more repos | `.docs/shared/…`, never duplicated into both repo folders |
| ADR | `.docs/<repo>/adr/NNNN-<slug>.md` |
| Reference, finding, runbook, deferred TODO | `.docs/<repo>/notes/` or `.docs/shared/notes/` |
| Conventions for agents in one repo | `.docs/<repo>/agents/conventions.md` |
| Superseded material | `.docs/shared/archive/` |
| Code standard of a stack (what code review checks against) | `.docs/shared/standards/<stack>.md` (single repo: `.docs/<repo>/standards/<stack>.md`), with a `README.md` saying how rules are chosen |

When you move or rename a document, rewrite every reference to it, so links inside `.docs/` keep resolving.

### Tickets

```
.docs/<repo|shared>/issues/<feature-slug>/
├── spec.md              # problem, decisions (who/when), contract, out of scope, ticket table
├── release.md           # when it ships: PR + merge order, deploy order, manual migration SQL, post-deploy checks
└── issues/NN-<slug>.md  # one ticket per file, numbered from 01
```

Each ticket has, in order:
- title, `Status:` line, `Assignee:`, `Repo:` + branch, `Spec:` path, `Method:` (with the code standard file, if the
  stack has one);
- the task;
- `## When a test keeps failing` (attempt limit);
- `## Acceptance`;
- `## Do not`;
- `## Report back`{{REPORT_BACK}};
- `## Comments`.

Workers append their evidence under `## Comments` but do not commit `.docs/`; whoever reviews commits their edits with
the review. Reviews are appended as `### Review (YYYY-MM-DD) — accepted | returned`. Update `Status:` when a ticket
closes, and keep the spec's ticket table current.

### Auto-commit every `.docs/` change

`.docs/` is its own local git repo. Commit every change right away, on its own, without asking. This is the one
place agents commit by themselves{{COMMIT_POLICY}}.

```bash
git -C .docs add <paths> && git -C .docs commit -m "docs(<area>): <what changed>"
```

- Make one commit per logical change: a new spec, a ticket batch, a review, a moved note.
- `<area>` is the folder touched, optionally with the feature slug (`docs(shared/<feature>): review ticket 02, accept`).
- Run `git -C .docs status --short` first. If it shows changes you did not make, commit only your own paths and
  mention the others to the user.
- Never `push` and never rewrite history (`--amend`, `rebase`, `reset --hard`).

## 2. The shared feature doc in `<repo>/docs/`

Write it only **after the feature passes its final test** (e2e or the user's live check), one per repo the feature
touched:

- New file, named in the folder's existing style. Never edit, rename, move or delete an existing file in `docs/`,
  not even to fix a typo; colleagues own them, and history must stay intact.
- It explains the finished feature for other programmers and their agents:
  - behaviour and rules;
  - endpoints and screens;
  - code locations (`file:line`, checked against the current code);
  - tests and how to run them;
  - known gaps.
- `## Relation to existing docs`: include it only for real contradictions or retirements. Name the document, quote
  the statement, and say what changed.
- Self-contained: no ticket numbers, worker names, review history, or anything that needs `.docs/` to make sense.
- If the repo has no `docs/` yet, create it with that first file.

## 3. References: the one-way rule

Anything inside a repo may point to the repo's own `docs/`, never to `.docs/`.

| Inside a repo (code comment, test, commit message, PR text, `docs/` file, README, config) | Allowed? |
|---|---|
| `see docs/FEATURE_X.md` | yes |
| `.docs/…`, a ticket number or path from `.docs/` | **no** |
| An ADR number from `.docs/<repo>/adr/` | **no**; state the reasoning inline instead |

The reverse is fine: `.docs/` files may cite repo paths, commits and repo `docs/` freely.

**No workspace-local tooling either.** Commands that exist only in the user's workspace (a root `Makefile`, local
scripts outside the repos, personal skills) are never mentioned in repo files; describe the situation generically
instead ("after the local databases are reset"). Each repo's own scripts are fine.

When a comment needs the "why", write the reason in one or two sentences. If a longer explanation is worth sharing,
it belongs in the repo's `docs/`.

**Agents that do not load this skill** (other CLIs, worker panes) must get the rule in their brief: every ticket's
"Do not" says *"no `.docs/` paths or ticket numbers in repo files; never edit existing `docs/` files"*.

## 4. Checks before handing work back

Run in each repo touched:

```bash
# any mention of .docs/ in tracked + untracked (not ignored) files
(cd <repo> && git ls-files -co --exclude-standard -z | xargs -0 grep -nIE '(^|[^[:alnum:]_])\.docs/' ; true)
# docs/ changes must be only NEW files
git -C <repo> status --short -- docs/            # expect only "??" or "A " lines
```

Fail the review if:
- any repo file mentions `.docs/`;
- an existing `docs/` file changed;
- a `.docs/` or `.scratch/` folder was created inside a repo.

A hit in a file you did not touch is pre-existing: report it, do not fix it inside an unrelated task.

## Do not

- Put tickets, specs, plans, reviews or reports in `<repo>/docs/`.
- Create `.docs/` or `.scratch/` inside a service repo, or docs in a repo root or source folder.
- Add a new top-level folder under `.docs/` without updating `.docs/README.md` and this skill.
