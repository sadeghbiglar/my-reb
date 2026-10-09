---
name: h-dashboard-session
description: "Use when working in h-dashboard repo."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos]
metadata:
  hermes:
    tags: [h-dashboard, laravel, bootstrap, codegraph, git, session-start]
---

# h-dashboard Session Bootstrap

## When to Use
Any session touching `/home/runner/h-dashboard`: new feature, bug fix, refactor, test run,
or a plain structural question about the code. Run sections 1-5 before real work.

Repo: `/home/runner/h-dashboard` (also the configured Hermes `terminal.cwd`).
Branch: read with `git branch --show-current` (this server: `rebecca`). Canonical repo:
`sadeghbiglar/h-dashboard`, branch `beta`.
Only remote is `origin`; **never add, rename, or delete remotes.** Confirm what `origin`
points at with `git remote -v` every session — do not hardcode a fork URL.

## 1. Never
- Do not clone a second copy of the repo.
- Do not switch branches unless asked.
- Do not push dev work straight to `beta`.
- Do not hardcode a fork URL or branch name — read `git remote -v` / `git branch --show-current`.

## 2. Session start sequence
```bash
cd /home/runner/h-dashboard
git remote -v
git branch --show-current
git status
git fetch origin
git merge origin/beta     # resolve conflicts; never rebase published work
```
Push only after step 4 verification passes.

## 3. Read project rules first
Root `AGENTS.md` is authoritative — read before touching code and keep it in context.
It already documents CodeGraph, Boost/Context7/GitHub MCP, Pest, Playwright, conventions,
and the gotchas table. Do not re-derive them.

## 4. Verify tooling, never assume
```bash
codegraph status .        # must be up to date; else `codegraph sync`
hermes mcp list           # codegraph, laravel_boost, context7, github enabled
```
Fix anything broken before continuing. Never claim a tool was used when it wasn't.

## 5. CodeGraph first for structural questions
Before grep/glob/Read crawling to understand structure, call CodeGraph MCP
(`mcp__codegraph__codegraph_explore`) or the CLI:
```bash
codegraph explore "how does AccessService accessibleUnitIds resolve unit hierarchy"
codegraph query "HardwareAuditObserver" --limit 5
```

## 6. Before committing
```bash
git status
git diff
php vendor/laravel/pint/builds/pint --test    # NOT vendor/bin/pint (see below)
php vendor/bin/phpstan analyse --no-progress --memory-limit=2G
XDEBUG_MODE=off php artisan test
```
`vendor/bin/pint` is a >1 MiB PHAR; the shell tool's lifecycle guard refuses to
scan it and blocks the command. Invoke the build script by path instead. Same
works for `phpstan` (`php vendor/bin/phpstan`).

Clear commit message, then `git push origin <current-branch>`. A push rejected
`non-fast-forward` means `origin/<branch>` has commits you do not have — the
branch was pushed from another session or server. Fetch and merge, never rebase
published work, then push again:
```bash
git fetch origin
git log --oneline HEAD..origin/<branch>     # read what you are absorbing first
git merge origin/<branch> --no-edit
git push origin <current-branch>
```
The branch may track `origin/beta` while the push target is the server branch —
always pass the branch name explicitly instead of bare `git push`.

### A batch of patches hides signature drift — diff the signatures after
`patch` is fuzzy-matching, so an edit can silently drop a `= null` default or
flip `private` to `public` while still reporting success. After any multi-patch
pass, compare every touched declaration against the base ref:
```bash
git diff -- resources/ | grep -E "^[+-]" | grep -E "function|public |private |protected "
```
Anything in that output you did not intend is a regression, not a no-op. Also
re-check baseline entries and defaults that a caller relies on — a dropped
default turns `->call('someMethod')` into a container
`BindingResolutionException` at test time.

## 7. Pull requests
When the user says `pr`, open a PR from the current branch to `beta` of
`https://github.com/sadeghbiglar/h-dashboard`. **Do not merge** unless asked.

Two tooling pitfalls, both verified 2026-10-06 (issue #818 / PR #822):

- **Use `gh pr create`, not the GitHub MCP `create_pull_request`.** The MCP tool
  silently drops the required `base` argument — the call fails twice with
  "missing required argument(s): base" no matter how the payload is built, and
  the body never reaches the server. `gh` works:
  ```bash
  gh pr create --repo sadeghbiglar/h-dashboard --base beta \
    --head <current-branch> --title "..." --body-file /tmp/pr.md
  ```
  Write a long body to a scratch file and pass `--body-file`; inline
  `--body` mangles backticks and newlines.
- **No `--maintainer-can-modify` flag** on this machine's `gh` — it errors and
  prints usage. Omit it; the fork relationship already allows maintainer edits.
- Watch CI with `gh run watch <run-id> --repo sadeghbiglar/h-dashboard --exit-status`
  (run id from `gh pr checks <pr> --repo sadeghbiglar/h-dashboard`). `--repo` is
  required — without it the run id does not resolve from this fork's context.

### A Sanctum ability assertion is worthless unless the session is detached
`Laravel\Sanctum\Guard::__invoke()` checks the `web` guard **first**. If a session
still resolves, it returns a `TransientToken` whose `can()` is unconditionally
true and the stored `abilities` are **never read** — so asserting "this token
cannot POST /api/hardware" against a logged-in session yields **422 from
validation, not 403 from the gate**. The test proves nothing.

```php
$token = Livewire::actingAs($user)->test('some.component')->get('someToken');
$this->app['auth']->guard('web')->logout();
$this->app['auth']->forgetGuards();
// now the request authenticates from the Bearer token ALONE
```

This is the same artifact that made issue #840's own "token survives logout"
measurement unreproducible. It is also the real threat model when a plaintext
token sits in rendered HTML: whoever reads it uses it with no session of their own.

Table name is **`hardwares`**, not `hardware`, for `assertDatabaseMissing`.

### `#[Locked]` throws in tests; it does not return 419
`CannotUpdateLockedPropertyException::render()` maps to a 419 response **only when
`app.debug` is off**. The test env has debug on, so `$component->set($locked, …)`
surfaces the exception. Assert it with `expectException`, not `assertStatus(419)`.
`#[Locked]` is also only *defence in depth* — it stops a client swapping a
server-written property, it does **not** remove a plaintext that is already
rendered into the HTML.

### Never run two suites at once — they share `h_dashboard_test`
`composer test` in the background while an interactive `php artisan test` runs
interleaves `RefreshDatabase` on the **same** database. Symptoms are misleading:
`drop table … cascade` failures, and unrelated tests failing that pass in
isolation. Kill the background run (`process_manage action=kill`) and re-run
serially — the results you got before the collision mean nothing.

### A test helper's override keys are often NOT fillable
`TicketsInboxLivewireTest::createTicket()` reads `$overrides['unit']` /
`['user']` for its defaults but then `array_merge`s them into the create
array, where `Ticket`'s `$fillable` has only `unit_id` / `user_id`. Passing
`'unit' => $unit` therefore lands the ticket in `Unit::first()` **silently** and
the ticket appears to be out of the viewer's scope. Set the real columns.

Two more traps in the same family:
- `createUserWithUnit()` returns keys `user` / `unit` — destructuring
  `['creator' => …]` throws `Undefined array key`.
- `createUserOnUnit()` grants **no Spatie permission**, so a viewer built with
  it cannot open a permission-gated page and the component under test never
  runs. Also `accessibleUnitIds()` reads `session('current_unit_id')` first and
  only falls back to the pivot, so seed it or the scope is `[]`.

### `Todo::accessible()` / `Builder::accessible()` is a PHPStan error
The scope lives on `HasOrganizationalScope`, so neither the static nor the
`Builder` form resolves at level 6 (`undefined static method` /
`undefined method Builder::accessible()`). Use an explicit
`->whereIn('unit_id', app(AccessService::class)->accessibleUnitIds())` — which
is also the form that fails closed.

`$ticket->setRelation(...)` after `Ticket::query()->whereIn(...)->find(...)`
reports `Cannot call method setRelation() on stdClass` (the known `@mixin`
gotcha). Fix with `/** @var Ticket|null $ticket */` above the assignment — and
**do not** switch to `$ticket->task = …`: that routes through `setAttribute()`,
parking the model in `$attributes` where a later `save()` tries to write a
non-existent column.

### Editing a Livewire component shifts the line-keyed PHPStan baseline
Any edit inside `resources/views/livewire/tickets/⚡*.blade.php` moves line
numbers, so `phpstan-baseline.neon` entries stop matching and `composer phpstan`
reports ~27 `ignore.unmatched (non-ignorable)` errors that are pure line drift.
Fix the genuinely new errors first, then:
```bash
composer phpstan-baseline && composer phpstan   # must print "[OK] No errors"
```
Do not hand-edit the baseline, and do not assume every reported error is real —
separate your own from the line-shift noise first (`git stash` + rerun gives the
true pre-existing count).

To prove the regeneration **suppressed nothing**, diff the baseline's `message:`
lines as a multiset before/after and normalise the anon-component line key
(`…blade\.php\:\d+\:\:` → `…blade\.php\:\:\:`). The invariant is
`Counter(new) - Counter(old)` **empty** — entries may legitimately be REMOVED
(you fixed those errors), so a falling total is the good case, not a failure. A
plain `git diff` lies about this: it looks like dozens of additions when every
one is just `:4::` → `:7::`.

Extract the multiset with a regex, not a YAML parse (the payload is a PHPStan
regex literal in single quotes):
```python
re.sub(r':\d+::', '::', m) for m in re.findall(r"message:\s*'(.*?)'", s, re.S)
```
Save the Counter to JSON *before* regenerating; after regenerating it is gone.

### Never "fix" a Livewire PHPStan type error with a native param hint
Livewire resolves component method arguments through
`Livewire\ImplicitlyBoundMethod`, which delegates to
`Illuminate\Container\BoundMethod`. For a **scalar native type on a required
parameter with no default**, `BoundMethod::addDependencyForCallParameter()`
cannot bind it and throws `BindingResolutionException` at call time — so the
component passes PHPStan and breaks every test that calls it.

Declare Livewire component types in **PHPDoc only**:
```php
/** @param  int  $ticketId */
public function toggleTicketSelection($ticketId): void   // no `int` hint
```
Return types and property shapes are safe (`@return`, `@var array<int, int>`).
Generic collection returns go in `@return` — PHP does **not** accept
`LengthAwarePaginator<int, Ticket>` in a native declaration and it is a parse
error.

Also: a real `identical.alwaysFalse` on a param PHPStan now types as non-nullable
means the guard was dead code — drop the null branch rather than widening the
docblock.

### The green-baseline proof for a branch someone else left red
When a branch is behind `beta` and its last commits were pushed from another
session, the baseline may already be broken before you touch anything.
Establish the truth with a throwaway worktree rather than by stashing:
```bash
git worktree add /tmp/wt-<name> origin/<branch>
ln -s /home/runner/h-dashboard/vendor /tmp/wt-<name>/vendor   # vendor is gitignored
git -C /tmp/wt-<name> checkout -q origin/beta && php vendor/bin/phpstan analyse
git worktree remove /tmp/wt-<name> --force
```
Then separate the error classes by JSON output — count the ones containing
`Ignored error pattern` (line drift) versus the rest (genuinely new):
```bash
php vendor/bin/phpstan analyse --error-format=json > /tmp/ps.json
```
Fix the real errors first, regenerate, and only then claim green.

## 8. Reporting completed work / push location
When the user asks "what did you do" or "where did you push", answer from git,
never from memory or commit messages alone:
```bash
git branch --show-current
git remote -v
git log --oneline -N
git log --oneline origin/<branch>   # what is actually on the remote
git branch -r --contains <sha>      # which remote branches contain a commit
git reflog | head                   # what happened THIS session (clone/checkout/push)
```
- A fresh clone makes reflog start at the clone, so commits already on
  `origin/<branch>` predate the session — do not claim them as this session's work,
  and do not treat them as uncommitted.
- Work done by a *previous* session on this branch still shows as "already there"
  when you open a new one. Read the prior turn's own closing summary from
  `~/.hermes/state.db` (`messages` table, newest rows, `role='assistant'`) when
  the user's "do your own suggestion" points at advice offered in a session you
  cannot see. `session_search` returns nothing when the transcript was compacted
  away, so the DB is the fallback.
- Memory entries naming fixes/perf work may refer to other branches or servers —
  verify existence with `git log --all --grep=<term>` before citing.
- Answer "where pushed" with remote name, full URL, branch, and the short SHAs.
- Format the work summary for the user's preference: terse Persian, ticket-style
  bullets — عنوان تیکت / تغییرات (sha) / علت. No narrative, no filler, no tables.

## 9. Admin / access grants are unit-scope work, not role work
`admin` role grants no unit scope: `accessibleUnitIds()` reads
`session('current_unit_id')` → `user_units` pivot → `person.u_id`, then
`Unit::descendantIds()`. "Full admin" therefore means a pivot row on the ROOT
unit. The national code is `persons.n_code` (there is no `melli_number`
column), and `user` is linked to `person` by `n_code`, not by id.
Adding a second `user_units` row makes `ValidateUnitContext` redirect to
`/select-context` on every request — warn about that trade-off and let the user
choose. Full procedure, verification, and cache-invalidation commands:
`references/user-access-grants.md`.

### Eligibility, not access, is what usually bounds a picker
Widening the pivot does **not** change which options a dropdown offers. Many
pickers filter on capability flags plus `is_active` and carry no scope predicate
at all, so "I gave the account full scope and the list is still short" is
answered by reading the picker's own query, not by granting more units.

**Rule:** compute and report three counts separately — eligible (flags), in-scope
(`accessibleUnitIds()`), accepted (validation rule). A short list is almost
always the flag column, and the pivot change was irrelevant to it. Say so
explicitly rather than letting the grant imply it widened the picker.

## References
- `references/user-access-grants.md` — role/pivot grants, cache invalidation, verification.
- `references/page-audit-checklist.md` — standing checklist for reviewing a page/component for scope, auth, parity and dead-capability gaps.

## 10. Superpowers skills are mandatory
`superpowers` plugin installed at `~/.hermes/plugins/superpowers` (v6.4.2, 15 skills,
source `obra/superpowers`, installed with `--force` after manual entrypoint review on
2026-10-05). Load the process skill before acting:
brainstorming before plan mode, systematic-debugging for bugs, test-driven-development
before writing behavior, verification-before-completion before claiming done.
Invoke as `skill_view("superpowers:brainstorming")` etc.

The `using-superpowers` bootstrap is injected only on the **first turn** of a session
(`pre_llm_call` hook, `is_first_turn`). A session that compacted over its first turn
lost it — that is the known failure mode, not a broken install. `AGENTS.md` above
repeats the load-bearing rules so they survive compaction.

## 11. Docs and audit skills
- `read-the-damn-docs` (software-development/read-the-damn-docs) — read official/current
  docs before implementing against any third-party API or library. Context7 MCP for
  version-specific Laravel/PHP docs.
- `improve` — shadcn audit skill, audit-only, do NOT execute its findings unless asked.