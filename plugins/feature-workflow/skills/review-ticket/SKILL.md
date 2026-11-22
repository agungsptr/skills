---
name: review-ticket
description: Review one worker ticket's code change against the project's written code standard, the ticket itself (spec axis) and for bugs — limited to that ticket's own commits and files, never the whole branch. Use in feature-workflow stage 5 (i), or when the user asks to review a ticket ("review ticket 03", "/review-ticket <ticket path>"). Read-only; it never edits code.
---

# Review one ticket

Wraps two review skills and controls their input, because neither can be pointed at "just this ticket":

- `mattpocock-skills:code-review` — **Standards** axis (the project's code standard + a code-smell baseline) and
  **Spec** axis (does the change do what the ticket asked). On its own it diffs `<fixed point>...HEAD`.
- `/code-review` — correctness bugs. On its own it reviews the current diff, a branch or a PR.

In a task worktree several workers commit in parallel on disjoint files, so `HEAD` and the branch hold other
tickets' commits. This skill cuts the change down to **the ticket's commits, limited to the ticket's files**, and feeds
exactly that to both. Never edit those two skills — plugin updates overwrite them.

Usage: `/review-ticket <ticket path> [--commits <sha> <sha> …]`

## 1. Read the inputs

1. **Project config.** The feature-workflow config (path in `CLAUDE.md` under "Feature workflow config"). From it take
   the `## Code standards` table (stack → standards file → repos) and the default source branch.
2. **Ticket.** From the ticket file take: `Repo:`, the worktree path and branch (`wip/<task>`), the `Spec:` path, the
   `Code standard:` in `Method:`, the file list (`## Files you may edit / create`; older tickets: "Your files:" under
   `## Setup`), and the commit SHAs its `## Comments` mention.
3. **Standards.** The standards file for the ticket's repo (ticket's `Code standard:` wins over the config table),
   plus the project files that file links. No standards file for the repo → Standards axis runs on the smell baseline
   only (all **consider**), and the report says so; spec axis and bug hunt run as usual.
4. **Nothing to review?** A ticket that changes no code (docs only, release notes) skips this skill — the manager's
   normal stage 5 checks cover it. An e2e-spec-only ticket is reviewed like code, against its repo's standards file.

## 2. Cut the scope

**Shell state does not survive between tool calls** (each Bash call is a new shell). Start every block with the same
variable lines, keep the file list in a zsh **array** (an unquoted `$FILES` is not split and matches nothing), and never
name a variable `path` (zsh ties it to `$PATH`):

```bash
W=<worktree, or the repo when the worktree is gone>; T=<task>; N=<ticket number>
FILES=( <ticket files> )
COMMITS=( <the ticket's own commits, oldest first> )
R="${TMPDIR:-/tmp}/review-$T-$N"                      # fixed path, so a later call can find and remove it
```

**The ticket's own commits.** Take them from the ticket's `Commits:` line in `## Comments` (ticket template), or —
older tickets — from the SHAs the worker names as its own per task ("### Task 0 (`b349a15a`)"), or from
`--commits`. **Not** from pasted `git log` output, which lists other tickets' commits too. Confirm each one:

```bash
for c in "${COMMITS[@]}"; do git -C "$W" show --stat --format='%h %s' "$c"; done   # each touches only FILES
SRC=$(git -C "$W" merge-base wip/$T "<source branch>")                              # cross-check only:
git -C "$W" log --reverse --format='%h %s' "$SRC"..wip/$T -- "${FILES[@]}"          # also shows other tickets
```

- A commit that touches files outside `FILES` is a scope breach (stage 5 (a)): list those paths; they are not reviewed.
- A commit the worker reported that does not exist, or one of its own the log shows but the ticket does not name, is a
  finding — never silently added or dropped.

**Review copy** — one throwaway worktree holding exactly the ticket's own change, staged, on top of the parent of its
first commit. Both reviews read it, so commits of other tickets in between (same files, done in sequence) stay out:

```bash
BASE=$(git -C "$W" rev-parse "${COMMITS[1]}~1")
git -C "$W" worktree remove --force "$R" 2>/dev/null; git -C "$W" worktree add --detach "$R" "$BASE"
for c in "${COMMITS[@]}"; do
  git -C "$W" show --format= --binary "$c" -- "${FILES[@]}" | git -C "$R" apply --index -3 || { echo "apply failed at $c"; break; }
done
git -C "$R" status --short                            # only ticket files, all staged
git -C "$R" diff --cached --stat | tail -1            # the scoped diff; empty → report "no change in the ticket's files"
```

If a patch does not apply (another ticket in between changed the same lines), say so in the report and review those
commits one by one with `git -C "$W" show <sha> -- "${FILES[@]}"`; never fall back to `BASE..HEAD`.

**Re-review after a return** covers only the fix commits made since the last review (`COMMITS` = those), with the
review copy based on the commit before the first fix; the spec axis checks the points the return asked for.

**Clean up** at the end, also after a failure, then confirm:
`git -C "$W" worktree remove --force "$R"; git -C "$W" worktree prune; git -C "$W" worktree list | grep -F "$R"` must
print nothing (other tasks may have their own `review-…` worktrees — leave those alone).

## 3. Run the two reviews

1. **`mattpocock-skills:code-review`** with these arguments, in this wording:
   - fixed point `BASE`; **use the diff command `git -C <R> diff --cached` (the review copy) and the ticket's own
     commit list from step 2 instead of `<fixed point>...HEAD`**;
   - spec: the ticket file (and the `Spec:` path it names);
   - standards sources: the standards file and the project files it links — these live outside the repo, so pass
     them by absolute path.
2. **`/code-review`** run from the review copy `R` (its current diff is the scoped diff), default effort level.
   Ask it to report only lines in that diff, plus existing code the diff newly calls or whose contract it changes,
   and tell it that `BASE` is the reference for "what production had" — not today's source branch, which may have
   moved since the ticket (dry run 2026-11-19: a later squash made a pre-ship migration edit look like a shipped one).

If a skill cannot take the scoped input (it reviews more than the scoped diff), say so in the report and drop its
findings outside `FILES`; never widen the scope to make it work.

## 4. Sort and report

Keep only findings in the scoped diff, or in existing code **the change reaches** (a new caller of an old function, a
changed contract, a copied pattern) — mark those "outside range (reachable)". Drop everything else.

A standard conflict that the ticket, its spec or a recorded decision explicitly chose (e.g. "keep the v3 return
types") is **not** a return: list it under Consider as "spec decision vs <rule>" so the manager can raise it with the
user (standards `README.md`).

```markdown
### Code review — <ticket id> (<n> commits on <BASE>, <n> files)
Must (return): <rule id + name> — <file:line> — <one line>; …        (none → "none")
Bugs (return): <file:line> — <what breaks, for which input>; …
Spec: missing / partial / wrong / not asked for — <ticket line> — <file:line>
Consider (noted): <rule id or smell> — <file:line>
Outside range (reachable): <file:line> — <why the change reaches it> — return only if the ticket's behaviour depends on it
Scope: <files outside the ticket touched by its commits, or "clean">
Verdict: return | accept | accept with notes   ·   Metrics: result=<…> reason=<standards|worker-error|…|->
```

The manager appends this under the ticket's `### Manager review`; the verdict feeds stage 5 and the metrics row.

## Do not

- Edit code, commit, or switch the task worktree's branch — the review copy is the only place files change.
- Review the whole branch, `HEAD`, or other tickets' files "to be safe".
- Return a ticket for a **consider** finding or for old code the change does not reach.
- Leave the review copy behind.
