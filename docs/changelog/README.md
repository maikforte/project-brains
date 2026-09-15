# Changelog

History written for future sessions: what changed, and **why**. Nothing here ships to
users. The rules are CLAUDE.md rules 1–3.

---

## Feature folders

Every entry lives in the folder of the feature it mainly serves. **Only folders listed
here may exist.** Add a row (and a `PROJECT.md` entry) before a feature's first entry.

| Folder | Covers |
|---|---|
| `general/` | Cross-cutting work: setup, theming, build, CI, dependencies with a behaviour change, architecture and design decisions, the audit trail and permission model themselves |

### Naming a folder

- Same name as `src/features/<feature>/`, `docs/manual/<feature>.md` and the permission
  prefix (ARCHITECTURE.md → Feature naming).
- Lowercase kebab-case: `purchase-orders`, not `PurchaseOrders` or `purchase_orders`.
- Name the business capability, not a screen or a technology: `login`, not `login-page`
  or `supabase-auth`.
- Before adding a folder, check this list for a feature it belongs to. `register` and
  `login` can be separate; `auth` and `login` should not both exist.

---

## Reading history

Before touching a feature (CLAUDE.md rule 1):

1. List `docs/changelog/<feature>/`. Filenames start with the date, so they sort oldest to newest.
2. Read the 5 newest, plus older ones whose title touches the task.
3. Search the whole folder tree for `**Features:**` lines naming the feature.
4. Read `docs/manual/<feature>.md`.

---

## Writing an entry

1. Copy `_template.md` to `docs/changelog/<feature>/YYYY-MM-DD-short-slug.md`.
2. The slug says what happened, in a few words: `form-validation`,
   `login-rate-limit`, `password-reset-expiry`.
3. Fill in only the sections that apply.
4. Commit it together with the code.

Two entries on the same day in the same folder are fine. Give them different slugs.
