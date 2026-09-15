---
paths:
  - "supabase/**"
  - "src/lib/supabase*"
  - "src/lib/database.types.ts"
---

# Working on the database

Read before writing SQL:

1. `docs/architecture/migrations.md`: idempotent SQL, repairs, self-asserting migrations.
2. `docs/architecture/auth-permissions.md`: RLS and `has_permission()` on every table.
3. `docs/architecture/audit-trail.md`: `audit_row_change()` trigger on every business table.

Every migration that creates a table must, in the same file:

- enable RLS and add policies that call `has_permission()`
- attach the audit trigger
- end with `select public.assert_table_guards();`

Then regenerate `src/lib/database.types.ts` and add a PGlite test.
