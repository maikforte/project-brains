---
paths:
  - "**/*.test.ts"
  - "**/*.test.tsx"
  - "**/*.stories.tsx"
  - "src/test/**"
---

# Writing tests and stories

Read `docs/architecture/testing.md` first. In short:

- UI: React Testing Library, queried by role and label, not by class or test id.
- Services: the Supabase fake in `src/test/supabaseFake.ts`.
- SQL, RPCs, triggers, policies: PGlite with the real migrations applied (`*.sql.test.ts`).
- Bug fixes: the regression test must fail before the fix.
- Stories: every variant, plus loading, empty and error states.
- Never skip or weaken a test to make a change pass.
