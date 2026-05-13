# The brief

The most important thing `/ralph` produces is a two-paragraph prose brief. It's the contract every iteration reads.

If you can't write the brief in two paragraphs, the planning isn't done. Go back to the user with one more question or push back to narrow the scope.

## Example

```markdown
# User Dashboard

## What we're building

A logged-in dashboard at /dashboard for the existing Laravel app, served by a
new DashboardController. Three sections: profile editing (name, email, avatar),
activity feed (last 30 days of the user's own actions, paginated), and account
settings. UI follows the existing Tailwind components in resources/views/components/.

## What's out of scope

No admin views, no impersonation, no multi-user/team features. The activity
feed reads from existing audit_log entries; we are not adding new audit hooks.
Avatar upload uses the existing Spatie Media Library setup. No new storage code.

## Invariants

- All routes go through the existing `auth` middleware
- No changes to the User model migration; new fields go in a separate table
- Tailwind classes only; no new CSS files
- `php artisan test --compact` must pass on every commit

## Cadence

Planner runs every 5 iterations. Moderate scope with a few cross-cutting
concerns (validation, auth) that warrant a mid-run sanity check.
```
