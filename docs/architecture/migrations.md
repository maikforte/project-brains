# Migrations

All schema changes go through `supabase/migrations/`. Never change a shared database
by hand except for the super admin bootstrap (see `auth-permissions.md`).

---

## Naming

```
supabase/migrations/YYYYMMDDHHMMSS_short_description.sql
```

**List the folder before naming a new file.** Parallel branches claim timestamps
mid-session, and two files with the same timestamp, or a new file stamped before one
already merged, cause confusing ordering.

---

## Write against the schema as it is

A migration may run against a database that moved on since it was written (another
branch merged, a hotfix went straight to prod). So:

- **Idempotent DDL:** `create table if not exists`, `add column if not exists`,
  `create or replace function`, `drop policy if exists` before `create policy`,
  `drop trigger if exists` before `create trigger`.
- **Before letting a migration apply late, check nothing stamped after it redefines
  the same function or policy.** Out-of-order application goes wrong by replacing a
  newer definition with an older one.

---

## Data repairs

- **Scope by the rule being restored, never by hardcoded ids.**
  `where status = 'Paid' and paid_at is null` finds the broken rows at run time and
  is a no-op once they're fixed. A repair naming specific ids would, run late, rewrite
  rows that had legitimately changed since.
- **Assert the end state, not the row count.** A count check fails when the repair
  correctly finds nothing to do.
- Repairs are audited automatically (null actor = system). Say in the changelog entry
  what was repaired and why.

---

## Every migration asserts its own result

End each migration with checks that fail the deploy if the result is wrong:

```sql
select public.assert_table_guards();   -- RLS + audit trigger on every table

do $$
begin
  if exists (select 1 from public.orders where status = 'Shipped' and shipped_at is null) then
    raise exception 'Repair incomplete: shipped orders without shipped_at';
  end if;
end $$;
```

A migration that checks itself catches what a manual review misses.

---

## Check every referenced column exists

PL/pgSQL resolves column names **when the function runs, not when it's created**. A
trigger referencing a column that doesn't exist compiles cleanly and breaks the first
write that fires it.

Before applying a migration with functions or triggers, query
`information_schema.columns` for **every** column the code references, including on
tables it joins to, not only the table it fires on. Neither the type checker nor the
test suite catches this unless a test runs that exact path.

---

## New table checklist

Everything below goes in **the same migration file** that creates the table:

```sql
create table if not exists public.customers (
  id         uuid primary key default gen_random_uuid(),
  code       text not null unique,
  name       text not null,
  is_active  boolean not null default true,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

-- 1. Permission codes
insert into public.permissions (code, feature, description) values
  ('customers.view',   'customers', 'See customers'),
  ('customers.edit',   'customers', 'Create and edit customers'),
  ('customers.delete', 'customers', 'Delete customers')
on conflict (code) do update set feature = excluded.feature, description = excluded.description;

-- 2. RLS and policies (see auth-permissions.md)
alter table public.customers enable row level security;
drop policy if exists customers_select on public.customers;
create policy customers_select on public.customers for select to authenticated
  using ((select public.has_permission('customers.view')));
-- ... insert / update / delete policies

-- 3. Audit trigger (see audit-trail.md)
drop trigger if exists audit_row_change on public.customers;
create trigger audit_row_change after insert or update or delete on public.customers
  for each row execute function public.audit_row_change('code');

-- 4. Indexes for the filters and sorts the list screen uses
create index if not exists customers_name_idx on public.customers (name);

-- 5. Guard check
select public.assert_table_guards();
```

After the migration:

1. Regenerate types: `supabase gen types typescript --local > src/lib/database.types.ts`
2. Add or update a PGlite test for any function, trigger or policy logic
3. Write the changelog entry, listing the migration file and permission codes

---

## Destructive changes

Dropping a column or table, changing a type, or deleting data needs the user's explicit
confirmation in the conversation before the migration is written. Prefer expand, then
migrate, then contract across separate migrations: add the new column, backfill, switch
the code, and drop the old column later.

## Never in a migration

- Credentials, user ids, or `is_super_admin = true`
- Production-only data fixes that name specific records
- `security definer` functions without a permission check (see `auth-permissions.md`)
