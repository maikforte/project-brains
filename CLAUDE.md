# {{PROJECT_NAME}}

{{One paragraph: what this app does, who uses it, and what it must feel like to use.}}

- **How the code is built** (stack, folders, naming, dependency rules) is in `ARCHITECTURE.md`, loaded below.
- **What the app does** (features, business rules, entities) is in `PROJECT.md`.
- **Why things are the way they are** is in `docs/changelog/`, one folder per feature.

@ARCHITECTURE.md

---

## 1. Read history before touching a feature (REQUIRED)

Every feature has one name (see ARCHITECTURE.md → Feature naming), and that name
is its folder in `docs/changelog/`. Earlier sessions recorded what they changed
and, more importantly, why. Read that before changing anything, so decisions are
informed by the project's past instead of guessed.

Before editing code in a feature:

1. List `docs/changelog/<feature>/`.
2. Read the **5 newest** entries in full, plus any older entry whose title touches the task.
3. Search all of `docs/changelog/` for entries whose `**Features:**` line names this
   feature. Those are changes filed under another feature that also affected this one.
4. Read `docs/manual/<feature>.md` if it exists.
5. **If the history conflicts with the task, say so before writing code.** Examples:
   an approach listed under "Rejected" is being proposed again, the task depends on
   something under "Still open", or the task would undo a deliberate behaviour.

For cross-cutting work (theming, build, dependencies, auth plumbing, the audit
trail), also read the 5 newest entries in `docs/changelog/general/`.

---

## 2. Change history per feature (REQUIRED)

`docs/changelog/` is history written for future sessions. Nothing in it ships to users.

**After any significant piece of work, add one new file:**

```
docs/changelog/<feature>/YYYY-MM-DD-short-slug.md
```

- `<feature>` must be a folder listed in `docs/changelog/README.md`. A new feature
  gets added to that list and to `PROJECT.md` **before** its first entry. This stops
  `login/` and `auth/` from both appearing.
- Work that belongs to no single feature (theming, build, CI, dependencies with a
  behaviour change, architecture decisions) goes in `general/`.
- A change that spans features goes in the folder of the feature it mainly serves.
  Its `**Features:**` line lists every feature it touched, so rule 1 step 3 finds it.

**Write an entry for:** a new feature, a behaviour change, a non-trivial fix, a
schema or permission change, an architecture decision.

**Don't write one for:** a typo, a dependency bump with no behaviour change, or
anything its commit message already fully explains.

**Write the entry as part of the work,** in the same commit as the code. An entry
written days later from memory is worth much less.

**Say why, not just what.** The diff already records what changed. What it cannot
record is the reasoning, the options rejected, and what is still unresolved.

**Append-only.** If an entry turns out to be wrong, add a new entry that corrects it
and links to the old one. Never edit history.

---

## 3. Entry format

Copy `docs/changelog/_template.md`. Open with `# YYYY-MM-DD — Title`, then the
`**Features:**` line, then only the sections that apply:

| Section | Holds |
|---|---|
| What happened / Cause | The problem or request, and the root cause if it was a fix |
| Changes | Files, migrations, RPCs, permissions: what now works |
| Tests | Test files added or changed, or why there are none |
| Rejected | Options considered and why they lost |
| Still open | Known gaps, follow-ups, unverified assumptions |

Short, not an essay. Reference file paths, migration names and commit hashes where they help.

---

## 4. Tests with every change (REQUIRED)

A change is not done until it is tested in the same commit.

| Change | Required |
|---|---|
| New or changed behaviour | A unit test that covers it |
| Bug fix | A regression test that **fails before the fix** and passes after |
| Migration, RPC, trigger, policy | A PGlite test (see `docs/architecture/testing.md`) |
| New or changed component or page | Its `.stories.tsx` updated: variants plus loading, empty and error states |

The changelog entry's **Tests** section names the test files. If there are none
(docs or styling only), it says why.

Before committing, run lint, typecheck and tests. Fix failures your change
introduces. Do not silence or skip a test to get green.

---

## 5. User manual per feature (REQUIRED)

`docs/manual/<feature>.md` is written for end users. **Update it in the same
change** whenever you:

- add, remove or rename a page users navigate to
- change a workflow, a status lifecycle, or the order steps must be done in
- add, remove or rename a field users fill in, or change its validation
- change how a displayed figure is calculated
- change a default that alters what a screen shows on open

**Don't** update it for refactors, performance work, or styling that doesn't change
what a user does. Format and style rules are in `docs/manual/README.md`.

---

## 6. Follow the architecture and design docs

`ARCHITECTURE.md`, `docs/architecture/*.md` and `docs/design/ui.md` are the agreed
design. Follow them.

If a task genuinely needs to break one, **update that doc first** and write a
`general/` changelog entry saying why, then make the change. Never let the code and
the doc quietly disagree.

---

## 7. Permissions with every feature (REQUIRED)

A new feature declares, in the same change:

- its permission codes (`<feature>.view`, `<feature>.edit`, ...), inserted by migration
- RLS policies on its tables that call `has_permission()`
- `can()` checks in the UI that mirror those policies

Hiding a button is not security. The database must refuse what the UI hides.
Details: `docs/architecture/auth-permissions.md`.

---

## 8. Production data (STRICT)

**Never query production unless the user has asked for it in this conversation.**

- Default to local or UAT for everything: investigating a bug, checking a count,
  confirming a schema.
- If a question genuinely needs production data, ask first and say why UAT cannot answer it. Then wait.
- **Permission does not carry.** "Check prod for X" covers that query, not the next
  one, and not the rest of the session.
- This covers every route to production data: the SQL editor, a service key, `psql`,
  an Edge Function, the dashboard. The rule is about the data, not the tool.

---

## 9. Line endings

Write files with **LF** line endings. `.gitattributes` pins this on commit, but a tool
that rewrites a file in text mode on Windows still produces a diff where every line
looks changed.

---

## 10. Reuse before creating (REQUIRED)

Before creating a component, hook, helper or service function:

1. **Search** `src/components/ui/`, `src/lib/`, and other features' `index.ts` exports.
2. **Extend** anything close with a variant or a prop instead of copying it.
3. **Create** new only if nothing fits, and say in the changelog entry what was checked.
4. **Promote** a feature-level component to `src/components/ui/` the moment a second
   feature needs it. Never import another feature's internals to get it.

Details (variants, required files): `docs/design/ui.md`.

---

## 11. Audit every table and event (REQUIRED)

- A migration that creates a business table attaches the `audit_row_change()` trigger
  **in the same file**, and ends with `select public.assert_table_guards();`.
- A feature that exports, downloads or imports data calls `log_event()`.

Every change to data is recorded with who made it and the values before and after.
Details: `docs/architecture/audit-trail.md`.

---

## 12. Commits follow Conventional Commits (REQUIRED)

Format ([conventionalcommits.org](https://www.conventionalcommits.org/en/v1.0.0/)):

```
<type>(<scope>): <summary>

<body: why, not what>

<footer: BREAKING CHANGE / Refs>
```

**Type**

| Type | Use for |
|---|---|
| `feat` | New user-facing capability |
| `fix` | Bug fix |
| `perf` | Faster, same behaviour |
| `refactor` | Code change with no behaviour change |
| `test` | Tests only |
| `docs` | Docs only (manual, architecture, changelog-only corrections) |
| `style` | Formatting only, no code meaning change |
| `build` | Dependencies, Vite, tooling |
| `ci` | CI and deploy pipeline |
| `chore` | Anything else that doesn't touch `src/` behaviour |
| `revert` | Reverts a previous commit |

A migration takes the type of what it does for users (`feat` or `fix`), not a type of its own.

**Scope** is the feature name, the same as `src/features/<feature>/` and
`docs/changelog/<feature>/`: `feat(customers): ...`. For `general/` work, use a
specific area (`deps`, `audit`, `permissions`, `ui`) or leave the scope out.

**Summary**

- Imperative, lowercase, no full stop, 72 characters or fewer.
- Say the **outcome**, not the mechanism: `fix(invoices): stop two users paying the same invoice twice`,
  not `fix(invoices): update payInvoice`.

**Body** says why, wrapped at 72 characters. Point to the changelog entry for detail:
`See docs/changelog/invoices/2026-10-02-atomic-payment.md`.

**Breaking changes** get `!` after the scope and a `BREAKING CHANGE:` footer. Breaking
here includes a destructive migration, a removed or renamed permission code, and a
changed RPC signature.

**One logical change per commit.** The code, its tests and stories, the manual update
and the changelog entry go in the **same** commit.

**No AI attribution.** Commits and pull requests carry no `Co-Authored-By: Claude`
trailer and no "Generated with Claude Code" line. `.claude/settings.json` turns the
automatic attribution off (`"attribution": { "commit": "", "pr": "" }`). Don't add it by hand either.

```
feat(customers): let staff archive customers instead of deleting them

Deleting a customer broke every past order that referenced it. Archive
hides the customer from selection lists but keeps history intact.

See docs/changelog/customers/2026-10-02-archive-instead-of-delete.md
Refs: #42
```

---

## Done checklist

Before calling a change finished:

- [ ] History read (rule 1), and any conflict raised
- [ ] Tests and stories added or updated, and passing (rule 4)
- [ ] Manual updated, or not needed (rule 5)
- [ ] Permissions and policies in place for new tables or actions (rule 7)
- [ ] Audit trigger and `log_event()` calls in place (rule 11)
- [ ] Changelog entry written in the right feature folder (rules 2–3)
- [ ] One Conventional Commit holding code, tests, manual and changelog (rule 12)
