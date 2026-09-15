---
paths:
  - "src/components/**"
---

# Working on shared components

1. Read `docs/design/ui.md` (tokens, variants, required files) and
   `docs/architecture/testing.md` (tests and stories).
2. **Reuse before creating** (CLAUDE.md rule 10): extend an existing component with a
   variant or prop before adding a new one.
3. Every component folder has `Name.tsx`, `Name.test.tsx`, `Name.stories.tsx` and `index.ts`.
4. Components in `src/components/` never import from `src/features/` and never call Supabase.
