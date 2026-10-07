---
name: gh-ssh-server-workflow
description: "Use when porting an ssh.yml server workflow to a fork."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux]
metadata:
  hermes:
    tags: [github-actions, ssh, tailscale, hermes, server-provisioning]
---

# GitHub Actions ssh.yml Server Workflow

Each server fork (e.g. `sadeghbiglar/my-reb`, academy/rebecca instances) hosts a
`.github/workflows/ssh.yml` that checks out the project repo, installs PHP/deps,
starts Docker Postgres, Tailscale, Hermes gateway, and a keep-alive loop.

## When to Use
Porting an `ssh.yml` GitHub Actions server workflow from one project/fork to
another, or rotating the server repo / project repo it targets.

## Procedure for porting to a new project/fork
0. Work on the EXACT repo+branch the user named (URL or `owner/repo`). A
   similarly-named local clone may be a different server fork — check
   `git remote -v` before editing anything. If no local clone exists,
   `git clone --depth 1 -b <branch> https://github.com/<owner>/<repo>.git`
   under /tmp and work there; PAT-auth HTTPS push works from a fresh clone.
1. Find the local clone of the server repo: `/home/runner/work/<name>/<name>`
   (or the /tmp clone from step 0). Inspect: `git remote -v`,
   `git branch --show-current`, `ls .github/workflows`.
2. Copy the source ssh.yml over the target repo's `.github/workflows/ssh.yml`.
3. Substitute every project-specific token — checklist:
   - `REPO_ORIGIN` URL, `WF_NAME`, workflow `name:`, cron cadence
   - All `/home/runner/<project>` paths and `cp .env.example*` filename
   - `docker compose -f <file>` filename
   - Fallback PostGIS URL in the mcp_servers block (`postgresql://<db>:***@…`)
   - Step 16 sync target repo URL + step name/commit message/echo strings
   - PostGIS URL fallback must match the project's real DB name; check the
     target project repo for its actual `.env.example*`/docker-compose files
     before assuming `.env.example.pgsql` exists.
   Mandatory greps afterwards:
   - `grep -rn '<old-project>' .github/workflows/ssh.yml` → expect zero
   - `grep -n 'set-url origin'` line: URL must match the *server* repo, not the
     project repo (step 16 syncs hermes state back to the server repo).
4. Commit with the repo's identity and push to the **current** branch with an
   explicit refspec (`git push origin <branch>`).
5. Fresh clones on a new machine lack `git config user.name/email` — commits
   fail with "empty ident name". Set them locally in that clone first:
   `git config user.name "sadeghbiglar/"` and the noreply email, matching how
   the workflow itself configures identity.

## Pitfalls
- `grep` output in `sed -n`/`git diff` may display a truncated `${GH_T...git"`
  line — verify the full line with `python3 -c "print(repr(...))"` before
  assuming the workflow's PAT URL is correct.
- The fallback PostGIS URL in the mcp_servers block still names the old
  database (`h_dashboard`). It only applies when the project's `.env` is
  missing, but update it anyway so a fresh server never points at the wrong DB.
- `cp .env.example.pgsql .env` and `docker-compose-pgsql-.yml` filenames are
  project-specific — the target project may not have them at all (academy1
  ships only sqlite `.env.example`). Verify with `ls` on a fresh clone of
  the project repo BEFORE copying the workflow, or step 8/9 fail.
- After a mistake (wrong repo/file edited), `git revert` the commit and
  push — do not leave the bad state while deciding.
- Never overwrite another server instance's workflow: each server repo is
  isolated; porting means copying the file and editing, not symlinking.
