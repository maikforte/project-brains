# Data layer

How the app talks to Supabase. Each rule here prevents a specific, common failure.

---

## One client, one service per feature

- `src/lib/supabase.ts` creates the only client, typed with `Database` from
  `database.types.ts`.
- Each feature has `<feature>Service.ts`. **It is the only file in the feature that
  imports the client.** Pages and hooks call service functions.
- Services return typed data or throw. They never return Supabase's `{ data, error }`
  pair to callers.

```ts
// src/features/customers/customerService.ts
import { supabase } from '@/lib/supabase';
import type { Customer, CustomerListParams, Page } from './types';

export async function listCustomers({ page, pageSize, search }: CustomerListParams): Promise<Page<Customer>> {
  const from = page * pageSize;
  let query = supabase
    .from('customers')
    .select('id, code, name, is_active', { count: 'exact' })
    .order('code')
    .range(from, from + pageSize - 1);

  if (search) query = query.ilike('name', `%${search}%`);

  const { data, error, count } = await query;
  if (error) throw error;
  return { rows: data, total: count ?? 0 };
}
```

## Generated types

After **every** migration, regenerate:

```bash
supabase gen types typescript --local > src/lib/database.types.ts
```

Never hand-edit this file. A type error after regenerating is the point: it shows
every caller the schema change broke.

---

## Lists are paged on the server

**Never load a whole table into the browser.** Every list screen, export and
dashboard figure reads a page or an aggregate.

- Lists: `.range()` with a page size (default 50), filters and sort applied in the query.
- Dashboard figures: an RPC that returns the aggregate, not rows to count in JS.
- Exports: page through the query in chunks (for example 1,000 rows), never one giant select.
- Avoid N+1: use embedded selects (`select('*, customer:customers(name)')`) or an RPC,
  not one query per row.

**RLS makes deep OFFSETs expensive.** A policy runs on every row the scan touches,
including rows OFFSET throws away, so page 50 costs far more than page 1. For
large tables prefer keyset paging (`where created_at < last_seen order by created_at desc`)
or an RPC. Keep policies cheap (see `auth-permissions.md`).

`count: 'exact'` on a large table is its own full scan. Use `'estimated'` when an
approximate total is fine.

---

## Read-then-write goes in one RPC

If a write depends on a value the browser read earlier ("available = loaded − reserved"),
**it is a race.** Two users, or one user with a stale screen, will overwrite each other.
The typical result is the same thing booked or reserved twice.

Rules:

1. **The database computes the new value, not the browser.** Send what to change
   ("reserve 10 seats on event X"), never the resulting total.
2. **Multi-row or multi-table writes go in a Postgres function**, called with
   `supabase.rpc()`, in a single transaction. Either everything is written or nothing is.
3. **Lock what you check.** `select ... for update` on the rows whose values the
   decision depends on, in a consistent order (by id) to avoid deadlocks.
4. **Refuse, don't cap.** If the data changed and the request can no longer be met,
   raise an error. Silently doing less than the screen showed is worse.
5. **Signal "screen is out of date" distinctly**, with SQLSTATE `40001`. The service
   turns it into a typed `ConflictError`, and the page reloads the record and shows a
   toast naming what changed.

```sql
raise exception 'Record % changed since it was loaded', p_id using errcode = '40001';
```

---

## No caching of API responses

A service worker or HTTP cache must **never** cache Supabase requests. Stale figures
shown as current lead users to act on data that has already changed. If you add a PWA,
exclude `*.supabase.co` from every runtime caching rule. Client-side caching (for
example a query library) must refetch on focus and after mutations. Adopting one is a
`general/` changelog decision.

---

## Dates and times

- **Store `timestamptz`**, always UTC. Convert only for display.
- The business time zone is **{{Business time zone, e.g. Europe/London}}**. Anything
  judged "per day" (on time, today's totals, report ranges) uses that zone explicitly:
  `(occurred_at at time zone '{{zone}}')::date`.
- **Date ranges use explicit boundaries:** `>= start_of_day_in_zone and < start_of_next_day_in_zone`.
  Never `<= 'YYYY-MM-DD 23:59:59'`, and never `new Date('YYYY-MM-DDT23:59:59')`, which
  parses in the viewer's time zone.
- Date-only business fields (expiry, due date) are `date`, not `timestamptz`.
- Put date helpers in `src/lib/datetime.ts`, with tests.

---

## Errors

- Services throw. Pages catch at the action boundary and show a toast or an inline error.
- A permission failure (SQLSTATE `42501`, or an RLS refusal that returns zero rows on
  update) must surface as an error, not as silent success. **Check that an update
  affected a row:** `.select()` after `.update()` and treat an empty result as refused.
