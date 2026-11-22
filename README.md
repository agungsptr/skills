# agungsptr/skills

Claude Code skills, packaged as a plugin marketplace. Each skill is its own plugin, so you install only what you need.

## Install

```
/plugin marketplace add agungsptr/skills
/plugin install agt-setup@agungsptr
/plugin install feature-workflow@agungsptr
/plugin install multi-repo-doc@agungsptr
```

To update later, run `/plugin marketplace update agungsptr`.

## Plugins

| Plugin | Skills | What it does |
|---|---|---|
| `agt-setup` | `/agt-setup` | Sets up [AgentThread](https://github.com/agungsptr/AgentThread) (`agt`) for a project. It installs agt when it is missing, with your consent, then analyses the repos read-only. It proposes the whole `.agentthread/config.yaml` with evidence, asks one batch of questions and applies the config with `agt config apply`. |
| `feature-workflow` | `/feature-workflow`, `/review-ticket` | A coordinating "manager" agent takes a ticket, bug or feature from first discussion to a PR, working with worker agents. The flow covers the spec and tickets, worktrees and branches, reviewing worker commits, staged e2e, a release doc with manual migration SQL, and the squash for the PR. `review-ticket` reviews one ticket's change against the project's code standard, the ticket itself and bugs. |
| `multi-repo-doc` | `/multi-repo-doc` | Sets up "two doc homes" in a project or multi-repo workspace: a private, locally versioned `.docs/` at the root, and each repository's committed `docs/`. It also installs a project `manage-docs` skill, a `CLAUDE.md` block and a memory entry, so later sessions follow the rules. |

## Layout

```
.claude-plugin/marketplace.json     the marketplace (name: agungsptr)
plugins/<plugin>/.claude-plugin/    plugin manifest
plugins/<plugin>/skills/<skill>/    SKILL.md and its templates
```

## License

[MIT](LICENSE)
