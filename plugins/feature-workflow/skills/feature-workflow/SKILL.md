---
name: feature-workflow
description: Use when a coordinating ("manager") agent takes a ticket, bug or feature from first reading to a PR with worker agents — discussing the task, writing a spec and tickets, dispatching workers, creating worktrees and branches, reviewing worker commits, running staged e2e, writing a release doc with manual migration SQL, or squashing for the PR. Also when a project's feature workflow config must be set up.
---

# Feature workflow (manager + workers)

One task moves through ten stages. The manager coordinates and verifies; workers write the code; the user decides,
pushes and opens PRs. **Violating the letter of a rule is violating its spirit** — a deadline does not shorten the flow,
it only changes how many questions fit into stage 1. A small task compresses stages (one ticket, a short spec); a stage
is skipped only when the user says so, and the spec records it.

## Project config first

Every value that differs per project (branch names, commands, worker panes, migration tool, banned words) lives in the
project's config file, never in this skill:

1. Use the path the project's `CLAUDE.md` names under "Feature workflow config"; otherwise
   `grep -rl '^# Feature workflow — project config' .docs` from the workspace root.
2. Missing → create it from [templates/project-config.md](templates/project-config.md): fill what the repos show
   (branches, Makefile/package scripts, migration tool), ask the user once for the rest, add the pointer to `CLAUDE.md`.
3. Read it at the start of every task. Where it and this skill differ, the config wins.

Private docs (`.docs/`) and the repo's shared `docs/` follow the project's doc rules (`manage-docs` skill if present):
nothing in a repo may point at `.docs/`.

## Roles

| Who | Does | Never |
|---|---|---|
| Manager | private docs only (notes, spec, tickets, `release.md`, committed per the project's doc rules), worktrees, reviews, re-running tests, read-only DB queries, restarting the stack, browser check, `release.md`, the squash commit | edit repo files (even a one-line fix — send the worker the exact change), push, open PRs |
| Worker | edits **only the files its ticket lists**, commits them in the task worktree with a pathspec, reports back | push, amend, rebase, reset, stash, `git add -A`/`.`/`commit -a`, touch servers, edit the user's checkouts |
| User | decisions, source branch, push, PRs, running manual migration SQL | — |

Commit messages (worker and squash) are plain conventional commits: no AI attribution or session trailers, no ticket
ids or private-doc mentions — this overrides any harness attribution reminder. Add the config's banned words.

## Git: one worktree per task per repo

- Ask the user the **source branch** before creating anything (default from config, usually production).
- Manager creates, per repo the task touches: worktree `<worktrees>/<task>/<repo>`, branch `wip/<task>` from the source
  branch (config's worktree command). **Not one worktree per ticket** — tickets of one task share it.
- Parallel workers in one worktree only on **disjoint file sets**; a file two tickets need belongs to one of them, or
  they run in sequence. A worker needing an unlisted file stops with BLOCKED.
- **Never move or rename a worktree a worker session has used** — agent sessions bind to the folder. Create it at its
  final path.
- Stage 10: manager creates `feat/<task>` from the source branch, `git merge --squash wip/<task>`, one plain commit.
  `wip/<task>` keeps the per-ticket history and is not pushed.

## The ten stages

1. **Task discussion.** Explain the task in plain words; compare with code (`file:line`), private notes, shared docs,
   designs and (read-only) production data. Go story by story: current behaviour → asked → gap → options with a
   recommendation → user decides. Record each as `Dn (user|manager, date)` and commit the notes. Go down to file
   level. Design here: e2e cases (what must differ for the test to prove something), the browser checklist per role and
   viewport, and the shared docs the change contradicts.
2. **Spec** `<docs>/issues/<task>/spec.md` from [templates/spec.md](templates/spec.md): rules, decisions, API contract
   **read from the controllers/routes**, out of scope, worktree table, ticket table, "Manager decisions (correctable)".
3. **Tickets** `issues/NN-<slug>.md` from [templates/ticket.md](templates/ticket.md), each with its **files it may
   edit/create**, plus a file matrix deciding parallel vs sequence. Every BE ticket that changes a save/delete path has a
   service-level flow test using the exact payload the FE sends; the last ticket of a dependent chain has the
   cross-ticket flow test. A ticket that changes data ships the migration file and its test; the manager derives the
   manual SQL from it in stage 10. The e2e and docs tickets exist from the start; docs run after stage 8.
4. **Delegation.** Dispatch message, always the same shape:
   `Ticket: <path>. Worktree: <path> on wip/<task>. Your files: <list>. Also working here: <worker → files> — do not
   touch them. Read the ticket fully; report back with the command at its end.` Answer implementation questions yourself; escalate only product, scope,
   data or security decisions. Idle without a report → read its pane.
   **Every new ticket starts in a fresh worker session, and the manager proves it** (the config names the command):
   sending a "new session" command is not proof — check that the session id changed and the new session holds only
   the dispatch message. Returns 1 and 2 of the same ticket stay in the same session; what happens after that is in
   stage 5. Record the dispatch time for the metrics row.
5. **Review per commit:** (a) `git show --stat <sha>` ⊆ ticket's files; (b) diff matches spec and decisions; (c) re-run
   the ticket's tests yourself; (d) red-first proven for a bug fix; (e) no banned mentions in repo files; (f) behaviour
   beyond the ticket written down as "accepted with note"; (g) acceptance checks met honestly — a name split or built at
   run time to slip past a grep is a return, and a too-broad grep is fixed in the ticket, not worked around; (h) every
   result the report claims carries its own summary line (suite, counts) and matches the code — a report that only
   quotes a filtered run (e.g. tagged cases) has not run the rest; (i) **code review** — for every ticket that changes code,
   run the `review-ticket` skill on the ticket (standards file from the config; none → smell baseline only): it reviews only the ticket's own commits and files
   against the standard and the ticket (spec axis) and hunts bugs in the same scope. A **must** breach or a real bug is
   a return citing the rule or the bug; **consider** findings are noted; old code outside the ticket only where the
   change reaches it. Append `### Manager review (date) — accepted | returned`.
   Returned → same worker, new commit; small extras → follow-up `NNb`.
   **Return limit.** Returns 1 and 2 go back to the same session. If the ticket fails review a third time, do not
   return it again: evaluate in detail (ticket, spec, the worker's reports and `## Comments` history) and write down
   whether the cause was the ticket, the spec or the worker; fix the ticket or spec (spec decisions go to the user);
   have the old session write `### Handoff` (template) — or write it yourself if that session is gone; then dispatch
   the ticket to a **fresh session** (same proof as stage 4). A fresh session that fails again → the user decides
   (split the ticket, other harness or model, or take it over).
   A worker's `BLOCKED` after its attempt limit (ticket template) is a report, not a failure: read what it tried, answer
   or fix the ticket, and treat it like a return.
   **Metrics.** For every round (accepted, returned or blocked) append one row to the config's metrics file, with the
   fields that file defines (who, harness, model, session, times, wait, compaction, result, reason, size). Add the
   row number to the review heading: `### Manager review (date) — accepted | returned — metrics #N`. A bug from an
   accepted ticket found later (stages 6–8 or after release) goes into that ticket's rows as `escaped`.
6. **Combined verification:** full unit suite + type check once per repo in the task worktree.
7. **Staged e2e:** restart the stack from the task worktrees (config). Stage 2 = new spec(s); stage 3 = related suites;
   stage 4 = all suites — **only after the user confirms** (state scope and duration), on a known commit; re-run
   failures once and compare with the **saved baseline of the source branch**. A failure is "pre-existing" only if the
   baseline shows it **or** the same test, re-run once, also fails on the untouched source branch now (needed when a
   release landed after the baseline date: run exactly the "new" failures there before investigating them, then
   refresh the baseline). No baseline yet → ask the user to run the same suites on the untouched source branch first
   and save the result as the baseline.
8. **Browser check** by the manager: the stage-1 checklist per role and viewport, against the design; results in spec.
9. **Shared feature docs:** one new file per touched repo's `docs/` (never edit existing ones), with "Relation to
   existing docs" from stage 1; written and committed by the worker who wrote the code, reviewed like code.
10. **Release + handover:** `release.md` from [templates/release.md](templates/release.md) — what ships, PR targets and
    merge order, deploy order, **manual migration SQL**, post-deploy checks, rollback. Simulate every migration on a
    **copy** of a production restore (drop it afterwards) before calling it ready. Squash to `feat/<task>` and verify
    `git diff --stat wip/<task> feat/<task>` lists only files the source branch changed since the worktree was made
    (`git diff --stat <wip base> <source>`), then re-run the unit suites on the squashed tree; the user pushes and opens PRs. After merge, with the user's go-ahead: remove worktrees and `wip/<task>`, refresh the baseline.

## Manual migration SQL (when the user runs migrations by hand)

Per migration, in order: read-only preview (expected counts) → `BEGIN` → the exact `up()` statements, a backup table
for anything deleted or overwritten → insert the migration tool's "executed" row with its real columns (e.g. MikroORM `name, executed_at`; Django
`app, name, applied`), guarded by `NOT EXISTS`, so the tool won't run it again →
check queries with expected results → **stop before `COMMIT`** (`-- COMMIT;` / `-- ROLLBACK;`) → commented `down()`.
State whether it runs before or after the deploy and why (is the old code safe with the new data, and vice versa).

## Red flags — stop

| Thought | Reality |
|---|---|
| "Separate worktrees per ticket avoid conflicts" | Disjoint file lists do that; per-ticket worktrees create an integration ticket and late cross-ticket bugs. |
| "The release is tomorrow, skip the questions / browser check" | Rework costs more than a question. Ask fewer, never none. |
| "I'll just fix this one line myself" | Send the worker the exact change. |
| "Run the full e2e now, it's safest" | Stage 4 needs the user's yes and a baseline to compare against. |
| "This failure is probably flaky / pre-existing" | Only the baseline, the untouched source branch or an isolated re-run says so. |
| "The acceptance grep is clean" | Check how: a name split to dodge the grep is a return, not a pass. |
| "The worker says all e2e passed" | Ask for the summary line of each suite; a filtered run (tagged cases only) skipped the rest. |
| "I sent /new, so the session is fresh" | Prove it: the session id changed and holds only the dispatch. |
| "I'll pick the base branch" | Ask the user. |
| "Docs after merge" | Stage 9 ships in the same squash. |
| "Migration: they'll figure out the SQL" | `release.md` carries it, simulated, stopping before COMMIT. |
| "The worker said done" | A report is a claim; review the commit. |
| "The worker is still trying, it'll get there" | Three failed attempts on the same failure → it reports BLOCKED with what it tried. Read that. |
| "One more return to the same session will do it" | Two returns is the limit. Evaluate, then a fresh session with a handoff. |
| "I'll fill in the metrics later" | One row per round, written at review time; later means never, or from memory. |
| "Review the whole branch, to be safe" | Only the ticket's commits and files; the rest is someone else's ticket. |
| "The code around it does the same, so it's fine" | Existing code is not the standard. The config's code standard is. |
