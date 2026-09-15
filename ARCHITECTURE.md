# Architecture

How this project is built. This file is short on purpose and loads every session
through `CLAUDE.md`. Detail lives in the topic files listed at the bottom.

Changing anything here follows CLAUDE.md rule 6: update this file first, then write
a `docs/changelog/general/` entry saying why.

---

## Stack

| Layer | Choice |
|---|---|
| UI | React + TypeScript, built with Vite |
| Routing | React Router, with one route guard that checks `can()` |
| Styling | Tailwind CSS with semantic tokens only |
| Components | shadcn/ui as base primitives, `class-variance-authority` for variants |
| Icons | lucide-react |
| Backend | Supabase: Postgres, Auth, RLS, RPCs, Edge Functions |
| Unit tests | Vitest + React Testing Library |
| SQL tests | PGlite (Postgres in-process) |
| Component workshop | Storybook |

No other UI, theme, icon or state library without a `general/` changelog entry
explaining why.

---

## Folder layout

```
src/
├── app/                      # App.tsx, router, providers, route guard
├── components/
│   └── ui/
│       ├── primitives/       # shadcn/ui generated files (components.json "ui" alias)
│       └── Button/           # project shared components, one folder each:
│           ├── Button.tsx
│           ├── Button.test.tsx
│           ├── Button.stories.tsx
│           └── index.ts
├── features/
│   └── <feature>/            # one folder per feature
│       ├── <Feature>Page.tsx
│       ├── <Feature>Page.test.tsx
│       ├── <Feature>Page.stories.tsx
│       ├── <feature>Service.ts       # the ONLY file in the feature that calls Supabase
│       ├── <feature>Service.test.ts
│       ├── types.ts
│       └── index.ts                  # public exports only
├── lib/
│   ├── supabase.ts           # the single Supabase client
│   ├── database.types.ts     # generated, never hand-edited
│   ├── permissions.ts        # can(), loaded permission codes
│   ├── audit.ts              # logEvent() wrapper around the log_event RPC
│   └── ...                   # shared helpers, each with a .test.ts
└── test/
    ├── setup.ts              # RTL + jsdom setup
    ├── supabaseFake.ts       # in-memory fake for service tests
    ├── pglite.ts             # PGlite bootstrap: auth stubs + migrations
    └── fixtures/

supabase/
├── migrations/               # timestamped SQL, see docs/architecture/migrations.md
└── functions/                # Edge Functions (the only place the service role key lives)

docs/
├── architecture/             # topic files, see Index
├── design/ui.md              # visual design system
├── changelog/<feature>/      # history, see CLAUDE.md rules 1–3
└── manual/<feature>.md       # end-user manual, see CLAUDE.md rule 5
```

Larger features may add subfolders (`components/`, `hooks/`) inside their own
folder. The rules below still apply.

---

## Feature naming

**One kebab-case name per feature, used everywhere.** For a feature called `login`:

| Place | Path |
|---|---|
| Code | `src/features/login/` |
| History | `docs/changelog/login/` |
| Manual | `docs/manual/login.md` |
| Scope | the `login` entry in `PROJECT.md` |
| Permissions | `login.view`, `login.edit`, ... |

This single name is what lets a session find a feature's history, manual and
permissions without searching. A new feature is added to `PROJECT.md` and
`docs/changelog/README.md` before any code is written.

---

## Dependency rules

1. **Pages and components never call Supabase.** Only `<feature>Service.ts` and `src/lib/` do.
2. **Features never import another feature's internals.** Only what its `index.ts` exports.
3. **`components/ui/` and `lib/` never import from `features/`.**
4. **Something needed by two features moves down** to `components/ui/` or `lib/`
   (CLAUDE.md rule 10).
5. **Business rules that must hold under concurrency live in Postgres** (RPCs,
   constraints, triggers), not in the browser. See `data-layer.md`.

---

## Environments

| Env | Purpose | Database |
|---|---|---|
| Local | Development | `supabase start` |
| UAT | Client testing | {{UAT Supabase project}} |
| Prod | Live | {{Prod Supabase project}}, see CLAUDE.md rule 8 |

`.env` variable names (values never committed):

```
VITE_SUPABASE_URL=
VITE_SUPABASE_ANON_KEY=
```

`SUPABASE_SERVICE_ROLE_KEY` is **never** in the frontend `.env`. It exists only as an
Edge Function secret.

---

## Index

Read the topic file before working in its area. `.claude/rules/` sends you to the
right one automatically when you open matching files.

| File | Read when |
|---|---|
| `docs/architecture/data-layer.md` | Writing a service, a query, an RPC, or anything with dates |
| `docs/architecture/auth-permissions.md` | Adding a table, a permission, a policy, or a guarded action |
| `docs/architecture/audit-trail.md` | Adding a table, or a feature that imports or exports |
| `docs/architecture/migrations.md` | Writing any SQL migration |
| `docs/architecture/testing.md` | Writing tests or stories, or measuring performance |
| `docs/design/ui.md` | Building or changing any UI |
