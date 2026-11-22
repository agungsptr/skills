# Release — <task title>

Status: <ready — squashed on `feat/<task>`, waiting for the user's push and PR (date)>
Target: <branch>
Spec: [spec.md](spec.md)

## 1. What ships

| Repo | Worktree | Push this branch | Squash commit | Base |
|---|---|---|---|---|

- <per repo: behaviour, migrations, new shared doc, tests>

## 2. Squash (manager)

`git switch -c feat/<task> <source> && git merge --squash wip/<task> && git commit` per worktree. Verified: <tests on
the squashed tree; feat tree = wip tree + only what the source branch changed meanwhile>.

## 3. PR and merge (the user)

1. Review: `git -C <worktree> show feat/<task>`.
2. Push: `git -C <worktree> push -u origin feat/<task>` per repo.
3. PRs into <target>; merge order <and why>.
4. Deploy order: <migration SQL → backend → frontend, with the reason each step must come first>.

## 4. Migration SQL (<DB client>, production <db>)

Manual equivalent of `<MigrationName>`. <What it changes; what it never touches; whether the old code is safe with it.>
Preview on <restore date>: <counts>. Simulated on a copy of that restore: <result>.

```sql
-- 0. Preview (read-only). Expect <n>.
SELECT ...;

BEGIN;

-- 1. Backup of rows about to be deleted/overwritten (down() restores from it).
CREATE TABLE "<backup_table>" AS SELECT ...;

-- 2. The up() statements, exactly as the migration runs them.
...;

-- 3. Mark it executed so the migration tool does not run it again.
-- Use the tool's real columns (MikroORM: name, executed_at; Django: app, name, applied; …).
INSERT INTO <migration_table> (<columns>)
SELECT <values>
WHERE NOT EXISTS (SELECT 1 FROM <migration_table> WHERE <same key>);

-- Checks (expect: ...)
SELECT ...;

-- COMMIT;   -- when the checks look right
-- ROLLBACK; -- otherwise
```

### Rollback later (same as `down()`)

```sql
-- BEGIN;
-- ...
-- COMMIT;
```

## 5. After deploy — checks

1. <role, screen, expected result>

## 6. Rollback of the code

<Revert order; whether the migration can stay.>
