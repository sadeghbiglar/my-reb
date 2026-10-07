---
name: hermes-server-workflows
description: "Use when editing my-* ssh.yml server bootstrap workflows."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux]
metadata:
  hermes:
    tags: [github-actions, ssh-workflow, laravel, mysql, postgresql, docker]
---

# Hermes Server Workflow Adaptation

## When to Use
Editing a server bootstrap workflow in one of the `my-reb` / `my-red` repos — the
`ssh.yml` that GitHub Actions runs to set up PHP, Docker, Hermes, 9router, CodeGraph,
and a Laravel project on an ephemeral runner.

## Procedure
1. Check the repo's remote and branch before cloning:
   ```bash
   git remote -v
   git branch --show-current
   ```
   This server's working copy lives at `/home/runner/work/my-reb/my-reb` (`my-reb`,
   branch `rebecca`). `my-red` (branch `academy`) is the academy1 instance.
2. Read the project repo first. The workflow does `cp .env.example.<x> .env` and
   `docker compose -f docker-compose-<x>.yml up -d`. If those files do not exist in
   the project repo, add them there first — the workflow runs against the project's
   checkout, not the workflow repo.
3. Update the workflow as a linked set for any DB/driver change:
   - PHP extensions: `php8.5-mysql` for MySQL, `php8.5-pgsql` for PostGIS/pgsql.
   - Compose file name and `container_name` service names.
   - Env example file and `DB_*` values (DB_CONNECTION, port, database, user, password).
   - Do not keep PostGIS MCP wiring for a non-pgsql project.
4. Remove stale fallbacks: hard-coded `h_dashboard` / `h_dashboard` URLs in
   `PG_URL`/`mysql://` fallbacks must point at the actual project DB.
5. Commit and push the workflow repo on its own branch only.

## Pitfalls
- A `cp: cannot stat '.env.example.pgsql'` failure means the project repo lacks the
  file the workflow expects — fix the project repo, not just the workflow.
- `REDIS_PASSWORD` and `DB_PASSWORD` must both be present in the env example when the
  compose file references them; a missing variable fails `docker compose`.
- Pushing an academy1-flavored workflow to `my-reb/rebecca` overwrites this server's
  copy — always confirm `git remote -v` and target the right `my-*` repo/branch.
