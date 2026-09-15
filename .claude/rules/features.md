---
paths:
  - "src/features/**"
---

# Working in a feature folder

The folder name under `src/features/` is the feature name.

1. **Read its history first** (CLAUDE.md rule 1): the 5 newest entries in
   `docs/changelog/<feature>/`, entries elsewhere whose `**Features:**` line names it,
   and `docs/manual/<feature>.md`. Raise any conflict with the task before coding.
2. **Read `docs/architecture/data-layer.md`** before touching the service file or any query.
3. **Reuse before creating** (CLAUDE.md rule 10): search `src/components/ui/`,
   `src/lib/` and other features' `index.ts` before adding a component, hook or helper.
4. **Imports or exports data?** Call `logEvent()` from `src/lib/audit.ts` (CLAUDE.md rule 11).
5. **New action or screen?** It needs a permission code, a policy and a `can()` check
   (CLAUDE.md rule 7).
6. **Before finishing:** tests and stories, the manual page, and a changelog entry in
   `docs/changelog/<feature>/`, all in one commit: `<type>(<feature>): <summary>`
   (CLAUDE.md rule 12).
