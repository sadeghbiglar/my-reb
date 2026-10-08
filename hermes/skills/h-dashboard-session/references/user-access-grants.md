# Granting admin / org-wide access to a user in the database

Class of task: "make this user a full admin", "give this national code access to
everything", "why can't this admin see unit X". Pure DB work — no code change, so
nothing to commit or push.

## The core fact: `admin` role does NOT grant unit scope

`AccessService::accessibleUnitIds()` never looks at the Spatie role. It resolves
base units in this order:

1. `session('current_unit_id')` if set
2. else `$user->units()->pluck('units.id')` (the `user_units` pivot)
3. else `$user->person?->u_id`

then returns `Unit::descendantIds($baseUnitIds)` — a recursive CTE, so scope is
**the whole subtree below the attached units**.

Consequence: an account with role `admin` and all permissions can still see
**zero** units if `user_units` is empty, and can be missing the top of the org
tree if its pivot rows sit on a mid-tree unit. "Full admin" = admin role
(usually already present) **plus a pivot row on the ROOT unit**.

## Find the user — `n_code`, not a national-code column

The Iranian national code lives on `persons` as **`n_code`**. There is no
`melli_number` / `national_code` column; querying one fails with
`SQLSTATE[42703]: Undefined column`.

```bash
php artisan tinker --execute="
\$p = App\Models\Person::where('n_code','4400176134')->first();
echo \$p ? 'person u_id='.\$p->u_id.' '.\$p->f_name.' '.\$p->l_name : 'NO PERSON';
\$u = \$p?->user;
echo \$u ? PHP_EOL.'user id='.\$u->id.' roles='.\$u->getRoleNames()->implode(',')
          .' perms='.\$u->getAllPermissions()->count().'/'.\Spatie\Permission\Models\Permission::count()
  : PHP_EOL.'NO USER (person exists but no users row for that n_code)';
"
```

`User` is keyed to `Person` by `n_code` (`belongsTo`, and `Person` has the
matching `hasOne`), **not** by `user_id`. A missing `users` row means the person
cannot log in at all — grant the role and fix the link, don't hunt for a user id.

Then check scope and pivot together — this is the whole diagnosis:

```bash
php artisan tinker --execute="
\$u = App\Models\User::find(2);
echo 'pivot units: '.json_encode(\$u->units()->pluck('units.id')).PHP_EOL;
echo 'scope size: '.App\Models\Unit::descendantIds(\$u->units()->pluck('units.id')->all())->count()
  .' of '.App\Models\Unit::count().PHP_EOL;
foreach (App\Models\Unit::whereNull('parent_id')->get() as \$r)
  echo 'root '.\$r->id.' '.\$r->name.' desc='.App\Models\Unit::descendantIds(\$r->id)->count().PHP_EOL;
"
```

Compare `descendants of my pivot rows` against `Unit::count()` and list the
units *outside* the scope (`whereNotIn('id', descendantIds(...))`) — that diff
is the concrete "what you still can't see" answer for the user.

## Attach the root unit

`user_units` columns: `user_id`, `unit_id`, `role` (enum `responsible`|`staff`),
`is_primary`, timestamps; unique on `(user_id, unit_id)` → always `updateOrInsert`
or you hit a duplicate-key error. `id` is serial, so don't supply it.

```bash
php artisan tinker --execute="
DB::table('user_units')->updateOrInsert(
  ['user_id' => 2, 'unit_id' => 1],                       // 1 = root unit
  ['role' => 'responsible', 'is_primary' => false, 'created_at' => now(), 'updated_at' => now()]
);
app(App\Services\AccessService::class)->clearCache(App\Models\User::find(2));
"
```

Existing pivot rows are left alone — additive grants, never a replace.

**Clear the cache or the change is invisible.** `accessibleUnitIds()` is
`Cache::remember`-ed for 30 min under a versioned key. `clearCache($user)`
increments the `unit_hierarchy` version, which orphans every versioned
`accessible_units:` key at once — strictly better than `Cache::forget`-ing one
guess at a key. Also `unsetRelation('units')` after a pivot write inside the
same process, or the already-loaded relation hides the new row.

## Verify against the real resolver, not against the pivot

Pivot rows are necessary but not sufficient — `session('current_unit_id')`
wins over them. Exercise the actual code path under each context:

```bash
php artisan tinker --execute="
\$u = App\Models\User::find(2);
auth()->login(\$u);
foreach ([1, 3, null] as \$uid) {
  \$uid === null ? session()->forget('current_unit_id') : session(['current_unit_id' => \$uid]);
  app(App\Services\AccessService::class)->clearCache(\$u);
  echo 'current_unit_id='.var_export(\$uid, true)
    .' -> accessible='.count(app(App\Services\AccessService::class)->accessibleUnitIds(\$u)).PHP_EOL;
}
"
```

Success looks like `accessible=834` for every context (i.e. equal to
`Unit::count()`), not just for the new pivot row.

## Side effect you MUST warn about: the login flow changes

`ValidateUnitContext` (and `select-context`) branch on the **count** of
`user_units` rows: empty → `person.u_id`; exactly one → set it silently; **more
than one → redirect `/select-context`**, where the user must pick a unit before
landing on `/dashboard`. Adding a second row therefore pushes a previously
one-click admin through an extra page, and it re-appears on every request.

Offer the three resolutions explicitly rather than silently picking one:
- keep both rows and accept the `/select-context` step (full org visible, unit
  context still recorded for activity logs and tickets),
- move `is_primary` onto the root row so a sensible unit is preselected,
- delete the narrower row to get a single automatic context.

Any of these is a real behavioural trade-off for the user's logs and ticket
routing, so it is their call.

## Reporting

State it as: role/permission count, pivot rows before → after, and the verified
`accessible=NNN/NNN` under each `current_unit_id`. Include that the change is
DB-only (`h_dashboard` on the configured connection — read it from
`config('database.connections.pgsql.database')`, never assumed), so there is
nothing to commit or push, and say plainly that no code or file was touched.

## Related

Granting a **role** (not scope) is different: `assignRole('admin')` writes
`model_has_roles`, and the role's permissions come from `role_has_permissions`
via `RoleSeeder`/`PermissionSeeder`. An unknown permission name makes
`hasPermissionTo()` answer `false` silently rather than throw, so re-running the
seeders is how you sync a role whose permission set drifted.