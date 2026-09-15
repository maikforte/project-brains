# Testing

Every change ships with tests in the same commit (CLAUDE.md rule 4). This file says
which kind of test goes where.

---

## Layers

| What | Tool | File | Runs against |
|---|---|---|---|
| Pure logic (helpers, calculations, parsers) | Vitest | `name.test.ts` next to the code | Nothing |
| Components and pages | Vitest + React Testing Library | `Name.test.tsx` next to the component | jsdom + mocked service |
| Services | Vitest | `<feature>Service.test.ts` | `src/test/supabaseFake.ts` |
| SQL: RPCs, triggers, constraints | Vitest + PGlite | `name.sql.test.ts` | In-process Postgres with real migrations |
| RLS policies | Local Supabase | `name.rls.test.ts` (optional project) | `supabase start` |
| Visual states | Storybook | `Name.stories.tsx` next to the component | Browser |

**Tests sit next to the code they test.** Shared helpers live in `src/test/`.

---

## Commands

Scripts every project defines in `package.json`:

| Script | Does |
|---|---|
| `npm test` | `vitest run`: all unit, service and PGlite tests |
| `npm run test:watch` | `vitest` |
| `npm run typecheck` | `tsc --noEmit -p tsconfig.app.json` |
| `npm run lint` | `eslint .` |
| `npm run storybook` | Storybook dev server |
| `npm run build-storybook` | Storybook build (catches broken stories) |

Run lint, typecheck and tests before committing.

---

## Components and pages (RTL)

- Query the way a user finds things: `getByRole`, `getByLabelText`, `getByText`. Not
  class names, and `data-testid` only as a last resort.
- Interact with `userEvent`, not `fireEvent`.
- Mock the feature's service module, not Supabase, in page tests.
- Cover: renders, each variant, user interactions, loading/empty/error states, and
  permission-gated controls (hidden when `can()` is false).

```tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { vi } from 'vitest';
import { CustomersPage } from './CustomersPage';
import * as service from './customerService';

vi.mock('./customerService');

it('shows an empty state when there are no customers', async () => {
  vi.mocked(service.listCustomers).mockResolvedValue({ rows: [], total: 0 });
  render(<CustomersPage />);
  expect(await screen.findByText(/no customers yet/i)).toBeInTheDocument();
});
```

---

## Services (Supabase fake)

`src/test/supabaseFake.ts` is a small in-memory stand-in for the query builder
(`from().select().eq().range()`, `rpc()`), created once per project and reused. Service
tests check:

- the right table, columns, filters, sort and range are requested
- Supabase errors are thrown, not swallowed
- an update that affected no rows is treated as refused
- SQLSTATE `40001` becomes `ConflictError`

---

## SQL (PGlite)

Postgres logic is tested against real Postgres, not mocks. PGlite runs Postgres
in-process, so these tests run in `npm test` with no Docker.

`src/test/pglite.ts` bootstraps a database per test file:

1. **Stubs Supabase's `auth` schema**, which PGlite doesn't have:
   ```sql
   create schema if not exists auth;
   create table if not exists auth.users (
     id uuid primary key, email text, raw_user_meta_data jsonb default '{}'
   );
   create or replace function auth.uid() returns uuid language sql stable as $$
     select nullif(current_setting('request.jwt.claim.sub', true), '')::uuid
   $$;
   do $$ begin create role anon;          exception when duplicate_object then null; end $$;
   do $$ begin create role authenticated; exception when duplicate_object then null; end $$;
   ```
2. **Applies every file in `supabase/migrations/` in order.** Tests run against the real schema.
3. Exposes `asUser(db, userId)`, which runs `select set_config('request.jwt.claim.sub', $1, false)`,
   so `auth.uid()` returns that user.

Keep the stubs in one shared file so every SQL suite uses the same copy.

**What to cover in an RPC test:** the happy path, each refusal (permission, conflict,
invalid input), that a refusal writes nothing (atomicity), and the concurrency case that
motivated the RPC, replayed in order.

**RLS caveat:** PGlite connects as a superuser, and superusers bypass RLS. Policy tests
grant the table to `authenticated`, run `set role authenticated`, act, then
`reset role` (confirmed working on PGlite 0.5.x). Functions that check
`has_permission()` themselves are testable without switching roles.

---

## Bug fixes

1. Write the test that reproduces the bug.
2. **Run it and see it fail.**
3. Fix the bug, then see it pass.
4. Name the test in the changelog entry.

A regression test that was never seen failing proves nothing.

---

## Storybook

Every component in `src/components/ui/` and every page in `src/features/` has a story.

- `title`: `Components/<Name>` or `Features/<feature>/<Page>`
- `tags: ['autodocs']`, with every prop documented in `argTypes`
- Stories for: every variant and size, disabled, **loading, empty, error**, long
  content and overflow, and a permission-restricted view where relevant
- Pages get their data from mocked services or story args, never from a live Supabase

```tsx
import type { Meta, StoryObj } from '@storybook/react';
import { StatusBadge } from './StatusBadge';

const meta: Meta<typeof StatusBadge> = {
  title: 'Components/StatusBadge',
  component: StatusBadge,
  tags: ['autodocs'],
  argTypes: { status: { control: 'select' } },
};
export default meta;
type Story = StoryObj<typeof StatusBadge>;

export const Neutral: Story = { args: { status: 'neutral', children: 'Draft' } };
export const Success: Story = { args: { status: 'success', children: 'Completed' } };
export const LongLabel: Story = { args: { status: 'warning', children: 'Awaiting approval from finance' } };
```

---

## Measuring performance

**Measure through a real signed-in session**, in the browser or with a client using a
user's JWT. Never through the service role or a database role that bypasses RLS.

A query timed at 50 ms through a role that bypasses RLS can take 8 s, or hit the
statement timeout, for a real user. A role that skips RLS can't measure a page that
doesn't.

Record before/after timings in the changelog entry.
