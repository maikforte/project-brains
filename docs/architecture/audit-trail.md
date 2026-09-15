# Audit trail

Every change to data is recorded: who did it, when, and the values before and after.
Events that don't change table data (exports, downloads, imports) are recorded too.

---

## Why triggers, not application code

An app quickly grows dozens of write paths. Logging from app code means every
one of them has to remember, and the one that forgets fails silently. Triggers:

- can't be forgotten once attached
- also catch changes that never go through the app: migrations, imports, manual fixes
  in the SQL editor. Those are the changes that are hardest to explain later.

What triggers can't see (a file download never touches the database) goes through
`log_event()`, called by the feature.

---

## The `audit_log` table

| Column | Holds |
|---|---|
| `id` | Sequence |
| `occurred_at` | When |
| `actor_id` | `auth.uid()`. **Null means "system"**: a migration, import or SQL-editor fix with no user session. No foreign key, so deleting a user never rewrites history. |
| `action` | `INSERT`, `UPDATE`, `DELETE` or `EVENT` |
| `table_name` | Table the row belongs to (null for events) |
| `record_id` | The row's key, as text |
| `record_label` | A human-readable label (`code`, `name`, ...) captured at the time |
| `changes` | See below |
| `event_name` | For `EVENT` rows: `customers.export`, `items.import`, ... |
| `details` | For `EVENT` rows: file name, row counts, filters used |

### What `changes` holds

| Action | `changes` |
|---|---|
| `INSERT` | The full new row |
| `UPDATE` | **Only the fields that changed**: `{ "name": { "from": "Acme", "to": "Acme Corp" } }` |
| `DELETE` | The full old row, so a deleted record can always be seen and restored |

- `created_at` and `updated_at` are ignored. `updated_at` moves on every write, so
  including it would turn every no-op into an event.
- **An update that changes nothing writes nothing.** Re-importing 8,000 identical rows
  creates zero audit rows.
- Sensitive columns are excluded per table through a trigger argument (see below).

---

## Rules

1. **Every business table gets the trigger in the migration that creates it.**
2. **Every migration ends with `select public.assert_table_guards();`**, which fails the
   deploy if any `public` table lacks RLS or the audit trigger. Adding a table to its
   exemption list is a `general/` changelog decision.
3. **The log can't be edited.** Rows are written only by `security definer` functions.
   There is no insert, update or delete policy. Reading needs `audit.view`.
4. **Features that export, download or import call `log_event()`** (through
   `logEvent()` in `src/lib/audit.ts`) after the action succeeds.
5. **Logins** are already recorded by Supabase in `auth.audit_log_entries`. Don't duplicate them.
6. **Business event logs are separate.** A domain ledger (payments, balance
   changes) with its own columns such as amount and account is built by the feature
   that needs it. `audit_log` doesn't replace it.

---

## Attaching the trigger

```sql
create trigger audit_row_change
  after insert or update or delete on public.customers
  for each row execute function public.audit_row_change(
    'code'          -- label column (optional)
  );
```

Arguments, all optional and positional:

| # | Argument | Default |
|---|---|---|
| 1 | Label column | none |
| 2 | Excluded columns, as an array literal, e.g. `'{api_token,notes_private}'` | `'{}'` |
| 3 | Key column | `id` |

```sql
-- A settings table keyed by "key", with a secret column excluded:
create trigger audit_row_change
  after insert or update or delete on public.integration_settings
  for each row execute function public.audit_row_change('key', '{client_secret}', 'key');
```

The function reads the key and label through `to_jsonb(new)`, so a table without an
`id` column records a null key instead of failing. (A trigger written against `NEW.id`
on a table keyed by `key` compiles cleanly and fails on the first write.)

---

## Calling `log_event()`

```ts
// src/lib/audit.ts
import { supabase } from '@/lib/supabase';

export async function logEvent(eventName: string, details: Record<string, unknown> = {}) {
  const { error } = await supabase.rpc('log_event', { p_event_name: eventName, p_details: details });
  if (error) throw error;
}

// in a feature, after the export succeeds:
await logEvent('customers.export', { fileName, rowCount, filters });
```

Event names follow permission codes: `<feature>.<action>`.

---

## Showing the log

The audit screen is its own feature (`audit-log`), built when the project needs it.

- Newest first, limited by date range, paged on the server.
- An update shows the change itself: **Name** ~~Acme~~ → **Acme Corp**.
- Column names map to user-facing labels. Unmapped columns fall back to the raw name
  with underscores replaced by spaces.
- **Foreign keys are stored as ids.** The screen resolves them to names in one query
  per referenced table. **An id that can't be resolved shows as the raw id, never a
  blank.** The log is a record, and hiding a value is worse than showing an ugly one.
- Booleans show as Yes/No, nulls as —, and a null actor as **system**.
- Exports of the log use the same resolution, so a download reads the same as the screen.

---

## Foundation SQL

Goes in the foundation migration **after** the block in `auth-permissions.md`, because
the read policy calls `has_permission()`.

```sql
-- Table -------------------------------------------------------------------

create table if not exists public.audit_log (
  id           bigint generated always as identity primary key,
  occurred_at  timestamptz not null default now(),
  actor_id     uuid,
  action       text not null check (action in ('INSERT', 'UPDATE', 'DELETE', 'EVENT')),
  table_name   text,
  record_id    text,
  record_label text,
  changes      jsonb,
  event_name   text,
  details      jsonb
);

create index if not exists audit_log_occurred_at_idx on public.audit_log (occurred_at desc);
create index if not exists audit_log_record_idx on public.audit_log (table_name, record_id, occurred_at desc);
create index if not exists audit_log_actor_idx on public.audit_log (actor_id, occurred_at desc);

alter table public.audit_log enable row level security;

drop policy if exists audit_log_select on public.audit_log;
create policy audit_log_select on public.audit_log for select to authenticated
  using ((select public.has_permission('audit.view')));
-- No insert, update or delete policy: rows come only from the functions below.

-- Row trigger -------------------------------------------------------------

create or replace function public.audit_row_change()
returns trigger language plpgsql security definer set search_path = public as $$
declare
  v_label_col text   := tg_argv[0];
  v_excluded  text[] := coalesce(tg_argv[1]::text[], '{}') || array['created_at', 'updated_at'];
  v_key_col   text   := coalesce(tg_argv[2], 'id');
  v_row       jsonb;
  v_old       jsonb;
  v_new       jsonb;
  v_changes   jsonb  := '{}'::jsonb;
  v_field     text;
begin
  if tg_op = 'DELETE' then
    v_row := to_jsonb(old);
  else
    v_row := to_jsonb(new);
  end if;

  if tg_op in ('UPDATE', 'DELETE') then v_old := to_jsonb(old) - v_excluded; end if;
  if tg_op in ('INSERT', 'UPDATE') then v_new := to_jsonb(new) - v_excluded; end if;

  if tg_op = 'UPDATE' then
    for v_field in select jsonb_object_keys(v_new) loop
      if (v_new -> v_field) is distinct from (v_old -> v_field) then
        v_changes := v_changes || jsonb_build_object(
          v_field, jsonb_build_object('from', v_old -> v_field, 'to', v_new -> v_field)
        );
      end if;
    end loop;

    if v_changes = '{}'::jsonb then
      return null;  -- nothing that matters changed: write nothing
    end if;
  elsif tg_op = 'INSERT' then
    v_changes := v_new;
  else
    v_changes := v_old;
  end if;

  insert into public.audit_log (actor_id, action, table_name, record_id, record_label, changes)
  values (
    auth.uid(),
    tg_op,
    tg_table_name,
    v_row ->> v_key_col,
    case when v_label_col is null then null else v_row ->> v_label_col end,
    v_changes
  );

  return null;  -- AFTER trigger: return value is ignored
end;
$$;

-- Events ------------------------------------------------------------------

create or replace function public.log_event(p_event_name text, p_details jsonb default '{}'::jsonb)
returns void language plpgsql security definer set search_path = public as $$
begin
  if auth.uid() is null then
    raise exception 'log_event requires a signed-in user' using errcode = '42501';
  end if;

  insert into public.audit_log (actor_id, action, event_name, details)
  values (auth.uid(), 'EVENT', p_event_name, coalesce(p_details, '{}'::jsonb));
end;
$$;

revoke all on function public.log_event(text, jsonb) from public, anon;
grant execute on function public.log_event(text, jsonb) to authenticated;

-- Guard check: RLS + audit trigger on every public table -------------------
-- Edit the exemption list only with a general/ changelog entry.

create or replace function public.assert_table_guards()
returns void language plpgsql as $$
declare
  v_exempt  text[] := array['audit_log'];
  v_no_rls  text;
  v_no_audit text;
begin
  select string_agg(c.relname, ', ' order by c.relname) into v_no_rls
  from pg_class c
  join pg_namespace n on n.oid = c.relnamespace
  where n.nspname = 'public' and c.relkind in ('r', 'p')
    and not c.relrowsecurity;

  select string_agg(c.relname, ', ' order by c.relname) into v_no_audit
  from pg_class c
  join pg_namespace n on n.oid = c.relnamespace
  where n.nspname = 'public' and c.relkind in ('r', 'p')
    and c.relname <> all (v_exempt)
    and not exists (
      select 1
      from pg_trigger t
      join pg_proc p on p.oid = t.tgfoid
      where t.tgrelid = c.oid and not t.tgisinternal and p.proname = 'audit_row_change'
    );

  if v_no_rls is not null then
    raise exception 'Tables without RLS: %', v_no_rls;
  end if;
  if v_no_audit is not null then
    raise exception 'Tables without audit trigger: %', v_no_audit;
  end if;
end;
$$;

revoke all on function public.assert_table_guards() from public, anon, authenticated;

-- Audit the foundation tables ---------------------------------------------

drop trigger if exists audit_row_change on public.profiles;
create trigger audit_row_change after insert or update or delete on public.profiles
  for each row execute function public.audit_row_change('full_name');

drop trigger if exists audit_row_change on public.roles;
create trigger audit_row_change after insert or update or delete on public.roles
  for each row execute function public.audit_row_change('name');

drop trigger if exists audit_row_change on public.permissions;
create trigger audit_row_change after insert or update or delete on public.permissions
  for each row execute function public.audit_row_change('code', '{}', 'code');

drop trigger if exists audit_row_change on public.role_permissions;
create trigger audit_row_change after insert or update or delete on public.role_permissions
  for each row execute function public.audit_row_change('permission_code', '{}', 'role_id');

select public.assert_table_guards();
```

---

## Tests

`supabase/tests` or `src/test/audit.sql.test.ts` (PGlite), covering the generic
function once:

- insert records the full row
- update records only changed fields, with `from` and `to`
- an update that changes nothing records nothing
- `updated_at`-only update records nothing
- delete records the full old row
- an excluded column never appears
- no session gives a null actor
- a table keyed by something other than `id` records the right key
- `log_event()` records the event and the actor, and refuses without a session
- `assert_table_guards()` fails for a table without RLS or without the trigger

Feature tests don't re-test the trigger. `assert_table_guards()` covers attachment.
