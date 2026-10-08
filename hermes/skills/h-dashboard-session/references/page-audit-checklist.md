# Page / Component Audit Checklist

A repeatable pass for "review this page — what is missing or broken?". Order matters:
cheap structural greps first, then verification runs. Report grouped as
**works / serious defects / missing capability**, severity-first, not file order.

## 1. Locate the code
Single-file Livewire components put the class inline in the blade file
(`new class extends Component`). There is no companion `Create.php`. Grep the
blade file directly. Then enumerate its `public function`s — that list *is* the
feature surface, and comparing it against a sibling page (inbox) exposes the gaps.

## 2. Scope: is the boundary access or eligibility?
Grep the component for the scope helper and for raw unit queries, then classify
each predicate:
- `can_receive_tickets` / `is_active` / capability flags → **eligibility**, deliberately cross-org.
- `accessibleUnitIds()` / `whereIn(unit_id, …)` → **accessibility**.
- **absent** on a picker or a search → a finding, not a neutral fact. Report the
  page that lacks it alongside the page that has it, so the inconsistency is visible.

A rule that checks only existence+flags while the picker filters the same flags is
consistent. A rule that adds `exists:units,id` while the picker scopes is the mismatch.

## 3. Auth on every egress, especially files
Uploaded files are the classic hole: a stored path rendered as `asset('storage/'.$path)`
serves through the web server with **no route, no middleware, no ticket check**.

**Rule:** for every upload path, find the serving side and ask who can reach it.
`store(…, 'public')` + a direct asset link = unauthenticated read of that path.
Check whether the symlink the asset URL depends on even exists — its absence is a
second finding (the feature is dead AND unsafe). Report both; a 0-row table is not
evidence the path is safe, only that nobody has hit it yet.

## 4. Dead capability: column exists, form has no input
For each domain column, grep consumers. A column the DB, the API and the reports all
read, that no form writes, is a **silently dead feature** — its reports return a
constant. Name the consumer that is currently guaranteed to compute a fixed value
(a count of overdue rows against a column nothing sets is always 0). This is a
stronger finding than a cosmetic gap and should be ranked above it.

## 5. Constraint vs. form vs. API
Compare the DB CHECK constraint's value set against the form's validation rule and
the API's rule, all three. Then check the *labels* map to the *values* in the order a
reader assumes (higher value = more urgent). A mismatch is one finding; a set that
includes values no surface offers is a second. Verify with
`pg_get_constraintdef(oid)` from `pg_constraint` rather than trusting the migration.

## 6. Normalization parity on search
If a text-normalizing trait exists in the project and any model hooks it in a
`saving` event, every `LIKE` search over a model that does **not** is a
falsely-empty result for real Persian input (Arabic ي/ك, ZWNJ, Persian digits).
Check the search side for the normalization helper too. State it as "typed with ك,
stored with ك/ي" rather than a vague search-bug claim.

## 7. Write-path robustness
- duplicate submit: `wire:submit` without `wire:loading.attr="disabled"` and no
  in-flight flag. A unique natural key does not help when it is random per request.
- files/disk writes outside the DB transaction → orphaned files on failure.
- dropdown that only closes on selection: no outside-click, no Escape.

## 8. Cross-path feature parity
Diff the component's action methods against the sibling page's. If one path accepts
attachments / links a task / notifies and a sibling does not, that is a finding. Also
check terminal-state interactions: a status that counts as "still open" in one place
and "closed" in another leaves a derived record permanently uncompletable.

## 9. Verify before reporting
Run the feature's own test file and report the real pass count. Then run the probes
that make each finding a number rather than an assertion: column listing,
constraint definition, scope count, flag counts, row count for the risky table.
**A finding with a measured count is credible; a finding phrased as "probably" is not.**