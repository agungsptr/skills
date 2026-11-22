---
name: multi-repo-doc
description: Set up the "two doc homes" documentation layout in a project or multi-repo workspace — a private, locally versioned `.docs/` at the workspace root (specs, tickets, plans, ADRs, notes; never shared) and each repository's own committed `docs/` for the team — then install a project `manage-docs` skill, a CLAUDE instruction block, and a Claude Code memory entry so every later session follows the rules. Use when the user asks to set up, bootstrap, or copy this documentation setup into a new or existing project, or to make Claude remember the `.docs` vs `docs/` rule.
---

# Set up the two doc homes

Documentation lives in exactly two places, with opposite audiences:

| | `.docs/` (workspace root) | `<repo>/docs/` |
|---|---|---|
| Audience | the user and their agents only | the whole team and their agents |
| Versioned | its own local git repo, no remote, never pushed | committed with the repo |
| Holds | specs, tickets, plans, ADRs, notes, reviews, runbooks | one finished-feature doc per feature |
| Existing files | edited freely | **never edited**; only new files are added |
| References | may cite repo paths and repo `docs/` | must never be referenced from any repo file |

This skill installs that setup. It is idempotent: re-running it fills gaps and never overwrites the user's files.

## Step 1 — Inspect, then confirm the plan

1. **Find the workspace root.** It is the directory the user is working in.
2. **Detect the layout:**
   - **Workspace:** the root is *not* a git repo and holds several sibling repos. List the direct child directories
     that contain `.git`.
   - **Single repo:** the root *is* a git repo (`git rev-parse --show-toplevel` equals the root).
3. **Look for existing pieces:** `.docs/`, `.claude/skills/manage-docs/`, `CLAUDE.md` / `CLAUDE.local.md`, and each
   repo's `docs/`. Note what already exists, so you merge instead of replacing.
4. **Ask the user once** (one question each, with a recommended default):
   - **Language** of files written to disk. Default: English. The chat language stays whatever the user speaks.
   - **Who commits** service repos. Default: the user commits; agents never commit in repos, but they do
     auto-commit `.docs/`.
   - **Agent workers** (yes/no): do they coordinate worker agents through tickets? The report command itself lives
     in the feature workflow config, not here — never write a pane or session id into this skill, ids change.

   Do not ask anything else; use the defaults below.

## Template placeholders

Fill every placeholder in the files under `templates/`; none may survive. Check with `grep -rn '{{' <written files>`.

| Placeholder | Value |
|---|---|
| `{{LANGUAGE}}` | language for files on disk (default `English`) |
| `{{REPOS}}` | the repo folder names, comma-separated, as backticked names |
| `{{TREE}}` | the actual `.docs/` tree you created, drawn like the layout in Step 2 |
| `{{SHARED_LINE}}` | workspace: ` plus \`shared/\` for cross-repo work, so cross-repo work has a single home`; single repo: empty |
| `{{REPORT_BACK}}` | with workers: ` (the report command and its "blocked" variant, copied from the feature workflow config at ticket-writing time — never a pane id written from memory)`; otherwise drop the `## Report back` item and this placeholder |
| `{{COMMIT_POLICY}}` | default: ` — service repos are committed by the user only`; otherwise the user's answer |

## Step 2 — Create `.docs/`

```
.docs/
├── README.md                  # from templates/docs-README.md
├── <repo>/                    # one per repo (single-repo layout: use the repo's name)
│   ├── agents/                # conventions for agents working in that repo (conventions.md)
│   ├── issues/                # one folder per feature: spec.md + issues/NN-<slug>.md
│   ├── specs/  plans/  adr/  notes/
└── shared/                    # workspace layout only: issues/ specs/ plans/ notes/ archive/
```

`standards/` (code standards per stack, see the `manage-docs` template) is **not** created up front; it appears when
the user writes a first standard.

- Create only the empty folders. Add a `.gitkeep` so git tracks them.
- Fill `README.md` from `templates/docs-README.md` (see "Template placeholders"); drop
  `shared/` for a single repo.
- **Single-repo layout:** `.docs/` sits inside the repo, so it must stay out of the repo:
  - append `.docs/` to `<repo>/.git/info/exclude` (local-only ignore);
  - **do not** edit the shared `.gitignore`.
- **Make `.docs/` its own local git repo:**
  `git -C .docs init && git -C .docs add -A && git -C .docs commit -m "docs: initial layout"`.
  Never add a remote.

## Step 3 — Install the project skill `manage-docs`

Write `.claude/skills/manage-docs/SKILL.md` from `templates/manage-docs-SKILL.md`, filling the placeholders.

- **Workspace layout:** `.claude/` sits at the workspace root, which is not a repo.
- **Single-repo layout:** `.claude/skills/` is usually committed. Since this skill names `.docs/`, put it in
  `.claude/skills/` only if the user agrees to share it; otherwise install it as a user-level skill named
  `manage-docs-<repo>`.

If `manage-docs` already exists, diff it against the template and propose additions; never replace it.

## Step 4 — Add the instruction block

Use `templates/claude-md-section.md`:
- **Workspace layout:** append it to the root `CLAUDE.md` (create the file if missing). The root is not shared.
- **Single-repo layout:** append it to `CLAUDE.local.md`, which Claude Code treats as personal and does not commit.
  Also make sure `CLAUDE.local.md` is in `.git/info/exclude`. Never put it in the committed `CLAUDE.md`: that file is
  a repo file, and repo files must not mention `.docs/`.

Skip the block if a `## Documentation` section with the same rules already exists.

## Step 5 — Add the Claude Code memory

Write one memory so future sessions in this project load the rule before touching docs.

- **Location:** use the memory directory named in your own system prompt ("You have a persistent file-based memory
  at …"). If there is none, use `$CLAUDE_CONFIG_DIR` (default `~/.claude`) +
  `/projects/<absolute workspace path with every "/" replaced by "-">/memory/`.
- **File:** `docs-two-homes.md`, from `templates/memory-docs-two-homes.md` with the placeholders filled.
- **Index:** add one line to `MEMORY.md` in that directory (create the file if missing):
  `- [Docs: two homes](docs-two-homes.md) — private .docs/ default; repo docs/ never edited, one new feature doc after final test`.
- If a memory with the same meaning exists, update it instead of adding a duplicate.

## Step 6 — Verify and report

Run these and show the results:

```bash
git -C .docs log --oneline -1                                   # .docs is its own repo, initial commit present
ls .docs                                                         # expected folders
# no repo file mentions .docs/ (run per repo):
(cd <repo> && git ls-files -co --exclude-standard -z | xargs -0 grep -nIE '(^|[^[:alnum:]_])\.docs/' ; true)
git -C <repo> status --short                                     # single-repo: .docs/ and CLAUDE.local.md not listed
```

Report, in the user's chat language:
- every file created or changed;
- the memory file path;
- any pre-existing repo file that already mentions `.docs/` (report it; do not fix it).

## Do not

- Commit, push, or edit anything inside a service repo, except `.git/info/exclude` in the single-repo layout.
- Edit, move, or delete existing files in any repo's `docs/`.
- Overwrite an existing `.docs/README.md`, `manage-docs` skill, CLAUDE block, or memory. Merge or propose instead.
- Add a remote to `.docs/`.
