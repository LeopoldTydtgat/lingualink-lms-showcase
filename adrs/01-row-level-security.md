# ADR 1: Row Level Security as the primary security boundary

**Status:** Adopted, in production

**Date:** 2026

## Context

The platform serves three user types (students, teachers, admin) sharing one Postgres database. Students must never see other students' data. Teachers must only see their own students, classes, and pay. A handful of columns (internal notes, pay rates, policy flags) must be invisible to some roles entirely.

The platform is operated by one person. Any security model relying on "remember to add the right filter in every query" would eventually fail, because a single forgotten `where` clause in any of hundreds of queries would leak data.

## Decision

Enforce access control at the database layer with Postgres Row Level Security, not in application code.

- RLS is enabled on every table. Policies define what each authenticated role can read and write, keyed to the requesting user's identity.
- Application queries run against the database as the requesting user. If the application layer has a bug, the database still refuses to return rows the user is not entitled to.
- Column-level grants strip sensitive fields (internal notes, pay rates, policy flags) from non-admin roles, so even a permitted row comes back without its restricted columns.
- Privileged multi-step operations (booking, cancellation, account changes) run as `security definer` database functions with explicit permission checks inside, and EXECUTE on those functions is revoked from all roles that should not call them.

## Alternatives considered

- **Application-layer checks only.** Every API route filters data itself. Rejected: one missed check is a breach, and a solo operator cannot review every query forever.
- **Separate databases or schemas per role.** Strong isolation but heavy duplication and painful cross-role features (a class involves a student, a teacher, and admin oversight simultaneously). Rejected as disproportionate.
- **A service role in the API with checks in middleware.** Centralises checks but means every query runs with full privileges; a single middleware bug exposes everything. Rejected: it inverts the fail-safe.

## Consequences

**Good**
- Defense in depth. Application bugs stop being data-leak bugs.
- Policies are written once per table, not once per query.
- Auditable: the entire access model can be read in one place, in SQL.

**Costs and sharp edges (learned in production)**
- **Missing policies fail silently.** RLS enabled with no policy returns empty results, not an error. Policies must be created in the same change that enables RLS.
- **Column grants fail silently too.** A `select('*')` against a table carrying any column-level revoke returns null rows with no error. The fix that stuck: explicit column lists on every sensitive table, and a verification query after every DDL change.
- **`DROP FUNCTION` + `CREATE` resets EXECUTE grants.** Postgres treats the recreated function as new, and platform defaults grant EXECUTE to client roles on new functions. Every function recreate is followed by an explicit re-revoke, by name, as a standing checklist step.
- **RLS is not free.** Policies run per query, so they are kept simple and indexed columns are used in policy conditions.

## What I would tell someone building this

Treat the database as the last line of defense and assume the application will eventually have a bug. Then test the model from the outside: log in as the least-privileged user and try to read what you should not. The silent-failure modes (empty results, null rows) mean a green build proves nothing; only adversarial reads prove the boundary holds.
