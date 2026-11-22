# Feature workflow — project config

Project values for the `feature-workflow` skill. The skill holds the process; this file holds what differs here.
Where the two differ, this file wins. Chat language: {{CHAT_LANGUAGE}}; files on disk: {{FILE_LANGUAGE}}.

## Repos and branches

| Repo | Production branch | Staging branch | PR order |
|---|---|---|---|
| {{REPO}} | {{PROD_BRANCH}} | {{STAGING_BRANCH}} | {{PR_ORDER}} |

- Default source branch: {{DEFAULT_SOURCE}} — still ask the user every task.
- Worktrees: `{{WORKTREE_ROOT}}/<task>/<repo>`; create with `{{WORKTREE_CMD}}` (env files + dependency install).
- Banned in repo files and commit messages (besides AI attribution, ticket ids, private-doc paths): {{BANNED_WORDS}}.
- Files never committed: {{NEVER_COMMIT}}.

## People and workers

- Manager: {{MANAGER_ID}}. Workers (default order): {{WORKERS}}. Fallback worker: {{FALLBACK_WORKER}}.
- Report back: `{{REPORT_DONE_CMD}}` / blocked: `{{REPORT_BLOCKED_CMD}}`. Tickets say: "run it with your shell tool —
  printing it is not reporting".

## Metrics

- File: {{METRICS_PATH}} (append-only table, one row per review round; columns and reason codes defined in it).
- Harness and model of each worker: {{WORKER_INFO_CMD}}. Session ids and dispatch time: {{DISPATCH_CMD}}.
- Token / timing data per session, for a later script: {{HARNESS_DATA_PATHS}}.

## Code standards

| Stack | Standards file | Repos |
|---|---|---|
| {{STACK}} | {{STANDARDS_FILE}} | {{STACK_REPOS}} |

Stage 5 (i) runs the `review-ticket` skill for every ticket that changes code. A repo without a standards file gets the
smell baseline (all **consider**) on the Standards axis; spec axis and bug hunt run as usual. Docs-only tickets skip it.

## Commands

| What | Command |
|---|---|
| Restart the stack from a task's worktrees | {{RESTART_CMD}} |
| Stack status | {{STATUS_CMD}} |
| Unit tests / type check per repo | {{UNIT_CMDS}} |
| E2E stage 2/3 (one spec) | {{E2E_SPEC_CMD}} |
| E2E stage 4 (full) | {{E2E_FULL_CMD}} |
| Saved baseline of the source branch | {{BASELINE_PATH}} |
| Read-only production data | {{PROD_READ_CMD}} |

## Migrations

- Tool and "executed" table: {{MIGRATION_TOOL}} / {{MIGRATION_TABLE}}.
- Production runs migrations: {{MIGRATION_POLICY}} (e.g. "by hand in a DB client before the deploy").
- Simulation: on a copy of {{PROD_RESTORE}}, dropped afterwards.

## Project rules the manager must read before analysing

{{RULE_FILES}}

## Deviations from the skill

{{DEVIATIONS}}
