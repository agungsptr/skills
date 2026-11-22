# NN — <REPO>: <what this ticket delivers>

Status: open
Assignee: <worker>
Repo: `<repo>`; worktree `<worktree path>`, branch `wip/<task>` (do not create branches or worktrees)
Spec: `<spec path>`
Method: <TDD / red-first for a bug fix>. Read first: <project rule files>. Code standard: <the config's standards file
for this repo's stack, or "none">.

## Files you may edit / create

- edit: `<path>`
- create: `<path>`

Others working in this worktree now: <worker → files>. Any file not listed here is off-limits — stop with BLOCKED.

## Task

1. <step, with `file:line` anchors and the decision ids it implements>
2. Service-level flow test with the exact payload the FE sends (for save/delete paths).
3. Run `<ticket's test command>` and `<type check>`; paste the output.
4. Commit only your files: `git add <new files> && git commit -m "<conventional message>" -- <your files>`.
   A new commit for each review fix.

## When a test keeps failing

- **Attempt limit:** if the same test still fails for the same reason after 3 attempts to fix it, stop. Write under
  `## Comments` what you tried and what each attempt showed, then send the blocked report with "attempt limit".
- You may correct a test **you wrote for this ticket** when the test itself is wrong; say why in `## Comments`. Never
  change an existing test or an acceptance check of this ticket to make it pass — that is a return, not a fix.

## Acceptance

- <observable result + the evidence to paste>
- `git show --stat HEAD` lists only the files above.
- No **must** rule of the code standard broken in your diff (when the ticket names one).

## Do not

- push, amend, rebase, reset, stash, `git add -A` / `.` / `commit -a`, delete `index.lock` (wait and retry).
- restart or stop servers (including build-tool daemons), edit the user's own checkouts.
- mention private docs, ticket ids, AI attribution or <banned words> in any repo file, test name or commit message.

## Report back

<!-- Use the config's report command and words (e.g. DONE/BLOCKED or the project's own). -->

Append evidence under `## Comments`, ending with one line naming **only your own commits for this ticket**, oldest
first (update it after each review fix): `Commits: <sha> <sha> …` — the review reads exactly these. Then **run it with
your shell tool — printing it is not reporting**:

```bash
<report-done command with "[<worker>] DONE NN: <summary>. Report: <ticket path>">
```

Blocked:

```bash
<report-blocked command with "[<worker>] BLOCKED NN: <what blocks you>. Report: <ticket path>">
```

## Comments

<!-- Handoff (written by the old session when the manager moves this ticket to a fresh session after two returns):
### Handoff (date)
- Done and committed: <commits, what they do>
- Tried and failed: <attempt → what it showed>
- Open: <what is left, what the reviews asked for>
- Assumptions to re-check: <what I believed that may be wrong>
The fresh session reads the ticket, this handoff, and `git log --oneline -10` in the worktree before changing code. -->
