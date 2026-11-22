---
name: agt-setup
description: Use when the user wants to install or set up AgentThread (`agt`) — "/agt-setup", "install agt", "install AgentThread", "set up agt", "configure agentthread here", "init agt for this project". Installs agt when it is missing (with consent), inspects the repos read-only, proposes the whole `.agentthread/config.yaml` (agents per role, repos, services, e2e, migrations, workflow facts) in one table with evidence, asks one batch of questions, then applies it with `agt config apply`. No manual config editing.
---

# Set up AgentThread for a project

Goal: from a workspace folder to a validated `.agentthread/config.yaml` with the user answering **one batch of
questions** and approving **one proposal**. Everything inferable is inferred, with evidence. A follow-up batch only when an answer changes other values. The config is written only
through `agt config apply`; never hand-edit `.agentthread/config.yaml`.

**Analysis only.** Apart from installing agt itself in Step 0 (with consent), setup never starts the stack, runs migrations, installs the project's dependencies, runs tests, or commits.

## Step 0 — Make sure agt is installed

Never use `sudo`. Never edit shell profiles without the user's consent. Never install anything silently: every clone,
link or install below needs the user's OK first.

1. Check: `command -v agt` and `agt --help` (must exit 0).
2. **agt is present** — see where it comes from: `ls -l "$(command -v agt)"` / `readlink`. If it resolves into a git
   clone, run `git -C <clone> fetch` and compare with upstream (`git -C <clone> status -sb`). When it is behind, *offer*
   (never force) an update: `git -C <clone> pull --ff-only && (cd <clone> && bun install)`. After an update, tell the
   user to restart: `agt daemon stop`, then re-run `agt start`. Not a clone, or up to date → go to Step 1.
3. **agt is missing** — check prerequisites first:
   - `bun --version` must be 1.1.34 or newer. Missing → tell the user the official install command from
     <https://bun.sh> and stop. Do not install Bun without explicit consent; with consent run
     `curl -fsSL https://bun.sh/install | bash` and note that Bun's bin dir may need to be on PATH.
   - `git --version`.
   - `command -v herdr`. Missing → tell the user agt needs Herdr (<https://herdr.dev>); continue setup, but warn that
     `agt start` will not work until Herdr is installed. Never install Herdr yourself.
   - At least one supported agent CLI: `command -v claude opencode codex omp agy`.
4. Ask **once** for consent and the install dir (default `~/.agentthread/AgentThread`). If that dir already exists and is
   a clone of the AgentThread repo, reuse it instead of cloning.
5. On consent:
   ```bash
   git clone https://github.com/agungsptr/AgentThread.git <dir>
   cd <dir> && bun install && bun link
   ```
6. Verify `command -v agt`. Not found → Bun's global bin dir (`~/.bun/bin`) is not on PATH: give the user the line to add
   to their shell profile (`export PATH="$HOME/.bun/bin:$PATH"`), do not edit the profile yourself, and use
   `~/.bun/bin/agt` for the rest of this session.
7. Return to the workspace folder (the install `cd` must not change where setup runs).

## Step 1 — Root and init

1. Confirm the cwd is the workspace root: a folder holding the git repos (children with `.git`), or a single repo. Not
   sure → ask. Never walk up or down on your own.
2. Run `agt init --no-setup` (non-interactive, a no-op when already initialised). Do not run `agt setup` (interactive wizard).

## Step 2 — Detect

```bash
command -v claude opencode codex omp agy     # installed agents → claude-code, opencode, codex, omp, agy
agt config show --json                        # current effective config; keep what is already set
```

Existing values are kept unless the analysis proves them wrong; say so in the proposal.

## Step 3 — Analyse (read-only, per repo)

| Area | Look at | Gives |
|---|---|---|
| Branches | `git branch -a`, `git symbolic-ref refs/remotes/origin/HEAD`, current branch, `.github/workflows`, `.gitlab-ci.yml` (branch filters, deploy jobs) | `source_branch`, `prod_branch`, `staging_branch`, `pr_order` (dependency order: API before UI) |
| Scripts | `package.json` scripts, lockfile (pnpm/yarn/npm/bun), `Makefile`, pyproject, go.mod | `unit`, `typecheck`, `install`, e2e commands, dev/start commands |
| Services | vite/next/nest/webpack configs, `PORT` / `--port` in scripts, `.env.example` keys, Dockerfile, docker-compose | candidate `services` with `start`, `port`, `health` |
| Health | the project's own scripts/Makefile (`curl`, wait/health targets) | `health` as the same `http(s)://…` URL; fall back to `tcp:<port>` only when none exists |
| Migrations | `.sequelizerc` (SequelizeMeta), prisma, knex, typeorm, mikro-orm, flyway, alembic, django | `migrate` per service, `migrate_on_start`, `workflow.migration.tool/table`, `prod_migrates_on_start` |
| Infra | docker-compose databases/queues | what the stack needs before services (`after`, or a service that starts them) |
| Reuse | existing `Makefile`, `.scripts/`, `scripts/` | stack commands that already exist — point config at them. Diff the project's own worktree script (env files copied, dirs like `e2e/.auth`, node_modules cloned/linked, install) against agt's built-in step: propose `copy` (files **and** directories), `link: [node_modules]` when the project already shares/clones node_modules, and `install`. Use `worktree_setup` only for what these cannot express |
| Env files | file **names** only (`.env`, `e2e/.env.local`), `.env.example` keys; env prefixes in scripts (`NODE_ENV=development yarn dev`) | `copy[]` per repo; service `env` map |
| Old notes | `.docs/**/feature-workflow*.md`, `CLAUDE.md` | browser URLs, login note, banned words, report words, read-first, never-commit |

Never open real `.env` or credential files by default; list names, read only `.env.example`. If the user explicitly allows
it (or offers to), grep only named non-secret keys (`PORT`, `*_HOST`, `*_PORT`, `NODE_ENV`, base URLs) from the real env
files; never read or print values of secrets (passwords, tokens, keys, DSNs with credentials). Take `file:line` or the command as
evidence for each value as you go.

## Step 4 — One proposal, one batch of questions

Present in a single message, in the user's language (detect it from their messages; reply in it throughout):

1. **Values table** — `key | proposed value | evidence | confidence (high/med/low)`. Cover the config areas below;
   omit keys with nothing to say (defaults apply).
2. **Services table** — `name | repo | dir | start | port | health | migrate | after`.
3. **Questions** — only what cannot be inferred, each with the inferred default: agent (+ model/effort) per role when
   several are installed, prod vs staging branch when ambiguous, chat/file language, report words, anything low
   confidence. Always include **migrations**: enable `migrate_on_start` (default **no**)? Say which DB `migrate` targets
   (from `.env.example`/config host) and warn when the host is not local. Also ask about tracked env files in `copy`
   (they become skip-worktree). Numbered, answerable in one reply ("all defaults" must work).

| Area | Keys |
|---|---|
| Agents | `agents.roles.{main,coordinator,worker}` (`agent`: claude-code, opencode, codex, omp, agy; `model`, `effort`); optional `agents.complexity.tiers` |
| Repos | `workspace.repos.<r>`: `path`, `source_branch`, `prod_branch`, `staging_branch`, `pr_order`, `unit`, `typecheck`, `copy[]` (files or directories; tracked files become skip-worktree), `link[]` (e.g. `node_modules`; skips `install` while lockfiles match), `install` |
| Services | `workspace.services.<name>`: `repo`, `dir`, `start`, `port`, `health` (`http(s)://…` or `tcp:<port>`), `env` (map: `NODE_ENV`, `PORT`… — never in `start`/`migrate` strings), `migrate`, `migrate_on_start` (default false), `after[]`, `enabled` |
| Worktrees | `workspace.worktree_setup` — only when the project has its own script (it replaces copy + install) |
| Workflow | `workflow.chat_language`, `file_language`, `stack.restart/status` (default to `agt stack` when services exist: leave unset), `e2e.stage2/3/4/baseline`, `migration.tool/table/prod_migrates_on_start`, `prod_read`, `browser.urls/login_note`, `banned_words`, `never_commit`, `read_first`, `report_words.done/blocked` |

Ports: when the chosen service port differs from the app's own config (`.env.example`, vite/next config, e2e base URL
in the Makefile/playwright), align `workflow.browser.urls` and the e2e base URLs/env with it, or ask. `agt config apply`
warns when a `browser.urls` port matches no service port, or a `copy` path is git-tracked; relay those warnings.

Constraints the schema enforces (get them right the first time): service names `^[a-z][a-z0-9-]*$`; every service's
`repo` exists in `workspace.repos`; ports unique among enabled services; `after` names known services, no cycles;
opencode `effort` needs a `model`; `install: ""` disables install.

## Step 5 — Apply

After the user approves or edits:

1. Write only the **partial** config (new/changed keys) to a temp file, YAML.
2. `agt config apply <file> --dry-run` → show the diff.
3. On confirmation: `agt config apply <file>` (a running daemon restarts unless `--no-restart`).
4. Validation error → fix the value in the temp file, re-run the dry run. Same error twice → ask the user. A project
   key is removed with `null`, never by editing the file.

## Step 6 — Finish

- `agt config show` and a short summary: roles, repos, services, what stays default.
- Services configured → mention the generated `Makefile` / `MAKEFILE.md` (not generated when there are no services).
- Next steps: `agt start` in the main tab (inside Herdr); `make start` or `agt stack start` to try the stack. Migrations
  run only on request: `agt stack start --migrate` / `make start MIGRATE=1` (or automatically with `migrate_on_start: true`).

## Red flags — stop

| Thought | Reality |
|---|---|
| "I'll just edit config.yaml" | Only `agt config apply`. |
| "Let me ask one thing at a time" | One batch, with defaults. Not a wizard. |
| "Port is probably 3000" | No evidence, no value: ask. Don't invent services or ports. |
| "I'll run the app / tests to check the port or command" | Setup is analysis only. Read configs and scripts. |
| "I'll peek at `.env` for the port" | Not without the user's explicit OK; then named non-secret keys only. Default: `.env.example` keys. |
| "Migrate on every start, it's convenient" | Default off; ask, and warn when the DB is not local. |
| "Put `NODE_ENV=…` in the command" | Prefixes go in the service `env` map. |
| "Their Makefile is messy, I'll replace it" | Keep project-owned scripts; point `worktree_setup`/`start` at them. |
| "Commit the config" | Never commit anything. |
| "Run the migration to find the table" | Infer from config files (e.g. SequelizeMeta); otherwise ask. |
| "Both prod and staging look plausible" | Ambiguous branch names are a question, not a guess. |
| "All values look obvious, skip the proposal" | The user approves one proposal with evidence before any apply. |
| "agt is missing, I'll just install it" | Ask once for consent and the install dir before cloning or linking. |
| "I'll use sudo / edit .zshrc so agt works" | No sudo. Never touch shell profiles without consent; print the PATH line instead. |
| "Bun or Herdr is missing, I'll install it" | Bun only with explicit consent; Herdr never. Point to bun.sh / herdr.dev. |
| "A newer agt exists, I'll update it" | Offer, never force; `--ff-only`, then tell the user to restart the daemon. |
