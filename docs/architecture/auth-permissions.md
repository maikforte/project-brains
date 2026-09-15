# Auth, roles and permissions

**Roles are data, not code.** The project starts with one super admin. Roles and
permissions grow feature by feature: each feature adds its permission codes, and the
super admin builds roles in the app by ticking those codes. No role list is fixed in
code.

Why: a role enum has to be migrated every time the organisation changes. With roles as
rows, "Managers can void, Editors can't" is one ticked box, not a schema change.

---

## The model

| Table | Holds |
|---|---|
| `profiles` | One row per `auth.users` user: `role_id`, `is_super_admin` |
| `roles` | Named roles, created in the app |
| `permissions` | The catalogue of permission codes, inserted **only by migrations** |
| `role_permissions` | Which codes each role has |

- **Super admin** bypasses permission checks. Set by hand, never by a migration.
- **A user with no role has no access.** `has_permission()` returns false. Fail closed.
- **Permission codes** are `<feature>.<action>`: `customers.view`, `customers.edit`,
  `customers.delete`, `orders.approve`. Add a specific code when an action is sensitive
  enough that some roles should have edit but not it (void, approve, release, delete).

---

## Day one: bootstrap

1. The foundation migration (below) creates the tables, functions and the foundation
   codes: `users.view`, `users.edit`, `roles.view`, `roles.edit`, `audit.view`.
2. Create the first user in the Supabase dashboard (Authentication → Users).
3. In the SQL editor, by hand:
   ```sql
   update public.profiles set is_super_admin = true where id = '<that user id>';
   ```
4. **Never** put credentials, a user id or `is_super_admin = true` in a migration. A
   committed migration is readable by everyone with the repo.

The super admin then creates roles in the app as features ship.

---

## Adding a feature's permissions

In the feature's migration:

```sql
insert into public.permissions (code, feature, description) values
  ('customers.view',   'customers', 'See customers'),
  ('customers.edit',   'customers', 'Create and edit customers'),
  ('customers.delete', 'customers', 'Delete customers')
on conflict (code) do update
  set feature = excluded.feature, description = excluded.description;
```

Then: policies on the feature's tables, `can()` checks in the UI, a line in the
`PROJECT.md` features table, and the codes listed in the changelog entry.

---

## Enforcement is in the database

**Hiding a button is not security.** Anyone signed in can call the API directly. Every
check the UI does, the database must also do.

### Policies

Every table has RLS enabled and policies per operation:

```sql
alter table public.customers enable row level security;

create policy customers_select on public.customers
  for select to authenticated
  using ((select public.has_permission('customers.view')));

create policy customers_insert on public.customers
  for insert to authenticated
  with check ((select public.has_permission('customers.edit')));

create policy customers_update on public.customers
  for update to authenticated
  using ((select public.has_permission('customers.edit')))
  with check ((select public.has_permission('customers.edit')));

create policy customers_delete on public.customers
  for delete to authenticated
  using ((select public.has_permission('customers.delete')));
```

- **Wrap calls in `(select ...)`.** Postgres then evaluates the function once per query
  (an InitPlan) instead of once per row. Without it, a policy on a 10,000-row table runs
  the user lookup 10,000 times, which can turn a 200 ms page into an 8 s one.
- **Never** write `using (true)` or `using (auth.uid() is not null)` on a write policy.
  That lets any signed-in user change anything. `assert_table_guards()` checks RLS is
  on, but reviewing policy bodies is still on you.
- If read access really is open to every signed-in user, say so in the changelog entry
  so it reads as a decision, not an oversight.

### SECURITY DEFINER functions

RLS does **not** apply inside a `security definer` function. Every such RPC checks
permission itself as its first statement:

```sql
if not public.has_permission('orders.approve') then
  raise exception 'Permission denied: orders.approve' using errcode = '42501';
end if;
```

### Column-level rules

A policy cannot tell *which* columns an update changed. When different permissions
govern different columns (edit a record vs. void it), add a `before update` trigger
that compares `old` and `new` and raises if the actor lacks the permission for a column
that moved.

---

## In the UI

- On sign-in, load the user's codes once with `rpc('my_permissions')` into
  `src/lib/permissions.ts`.
- `can('customers.edit')` gates buttons and actions. The route guard gates pages with
  `can('<feature>.view')`.
- Never check role **names** in code. Roles are data and can be renamed. Always check codes.
- Show the role's name from the `roles` row, never a hardcoded label.

---

## Foundation SQL

The first migration. Idempotent, so it is safe to re-run. The audit pieces it relies on
are in `audit-trail.md` and go in the same migration, after this block.

```sql
-- Profiles, roles, permissions --------------------------------------------

create table if not exists public.roles (
  id          uuid primary key default gen_random_uuid(),
  name        text not null unique,
  description text,
  created_at  timestamptz not null default now(),
  updated_at  timestamptz not null default now()
);

create table if not exists public.permissions (
  code        text primary key,
  feature     text not null,
  description text not null
);

create table if not exists public.role_permissions (
  role_id         uuid not null references public.roles(id) on delete cascade,
  permission_code text not null references public.permissions(code) on delete cascade,
  primary key (role_id, permission_code)
);

create table if not exists public.profiles (
  id             uuid primary key references auth.users(id) on delete cascade,
  full_name      text,
  role_id        uuid references public.roles(id) on delete set null,
  is_super_admin boolean not null default false,
  created_at     timestamptz not null default now(),
  updated_at     timestamptz not null default now()
);

-- A profile for every new auth user --------------------------------------

create or replace function public.handle_new_user()
returns trigger language plpgsql security definer set search_path = public as $$
begin
  insert into public.profiles (id, full_name)
  values (new.id, coalesce(new.raw_user_meta_data ->> 'full_name', new.email))
  on conflict (id) do nothing;
  return new;
end;
$$;

drop trigger if exists on_auth_user_created on auth.users;
create trigger on_auth_user_created
  after insert on auth.users
  for each row execute function public.handle_new_user();

-- Permission checks -------------------------------------------------------

create or replace function public.has_permission(p_code text)
returns boolean language sql stable security definer set search_path = public as $$
  select exists (
    select 1
    from public.profiles pr
    where pr.id = auth.uid()
      and (
        pr.is_super_admin
        or exists (
          select 1 from public.role_permissions rp
          where rp.role_id = pr.role_id and rp.permission_code = p_code
        )
      )
  );
$$;

create or replace function public.my_permissions()
returns setof text language sql stable security definer set search_path = public as $$
  select p.code
  from public.permissions p
  join public.profiles pr on pr.id = auth.uid()
  where pr.is_super_admin
     or exists (
       select 1 from public.role_permissions rp
       where rp.role_id = pr.role_id and rp.permission_code = p.code
     );
$$;

revoke all on function public.has_permission(text) from public, anon;
revoke all on function public.my_permissions() from public, anon;
grant execute on function public.has_permission(text) to authenticated;
grant execute on function public.my_permissions() to authenticated;

-- Only a super admin can grant or remove super admin ----------------------
-- auth.uid() is null in the dashboard SQL editor, which is how the first
-- super admin gets set.

create or replace function public.profiles_guard_super_admin()
returns trigger language plpgsql security definer set search_path = public as $$
begin
  if new.is_super_admin is distinct from old.is_super_admin
     and auth.uid() is not null
     and not exists (select 1 from public.profiles where id = auth.uid() and is_super_admin) then
    raise exception 'Only a super admin can change is_super_admin' using errcode = '42501';
  end if;
  return new;
end;
$$;

drop trigger if exists profiles_guard_super_admin on public.profiles;
create trigger profiles_guard_super_admin
  before update on public.profiles
  for each row execute function public.profiles_guard_super_admin();

-- Foundation permission codes --------------------------------------------

insert into public.permissions (code, feature, description) values
  ('users.view',  'users',     'See users'),
  ('users.edit',  'users',     'Create and edit users, assign roles'),
  ('roles.view',  'roles',     'See roles and their permissions'),
  ('roles.edit',  'roles',     'Create, edit and delete roles'),
  ('audit.view',  'audit-log', 'Read the audit log')
on conflict (code) do update
  set feature = excluded.feature, description = excluded.description;

-- RLS ---------------------------------------------------------------------

alter table public.roles            enable row level security;
alter table public.permissions      enable row level security;
alter table public.role_permissions enable row level security;
alter table public.profiles         enable row level security;

drop policy if exists profiles_select on public.profiles;
create policy profiles_select on public.profiles for select to authenticated
  using (id = (select auth.uid()) or (select public.has_permission('users.view')));

drop policy if exists profiles_update on public.profiles;
create policy profiles_update on public.profiles for update to authenticated
  using ((select public.has_permission('users.edit')))
  with check ((select public.has_permission('users.edit')));

drop policy if exists roles_select on public.roles;
create policy roles_select on public.roles for select to authenticated
  using (
    (select public.has_permission('roles.view'))
    or id = (select role_id from public.profiles where id = (select auth.uid()))
  );

drop policy if exists roles_write on public.roles;
create policy roles_write on public.roles for all to authenticated
  using ((select public.has_permission('roles.edit')))
  with check ((select public.has_permission('roles.edit')));

drop policy if exists permissions_select on public.permissions;
create policy permissions_select on public.permissions for select to authenticated
  using ((select public.has_permission('roles.view')));
-- No write policy on permissions: codes come from migrations only.

drop policy if exists role_permissions_select on public.role_permissions;
create policy role_permissions_select on public.role_permissions for select to authenticated
  using ((select public.has_permission('roles.view')));

drop policy if exists role_permissions_write on public.role_permissions;
create policy role_permissions_write on public.role_permissions for all to authenticated
  using ((select public.has_permission('roles.edit')))
  with check ((select public.has_permission('roles.edit')));
```

Profiles have no insert or delete policy: rows come from `handle_new_user()`, and users
are removed through an Edge Function that deletes the auth user.

---

## Edge Functions

Creating or deleting auth users needs the service role key, so it goes through an Edge
Function. The function:

1. reads the caller's JWT and calls `has_permission('users.edit')` **as the caller**
   (a client built with the caller's token), and refuses if false
2. only then uses a service-role client for the admin call
3. calls `log_event()` as the caller for anything triggers cannot see

The service role key exists only as an Edge Function secret.
