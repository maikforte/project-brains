# project-brains

A GitHub template for **React + Vite + Supabase** projects worked on with Claude Code.

It holds no application code. It holds the rules, architecture and history
conventions, so every new project starts structured the same way and every Claude
session reads what earlier sessions decided before changing anything.

> **Using this for a new project?** Follow [Start a new project](#start-a-new-project),
> then replace this README with your project's own (step 6).

---

## What's inside

| File | Purpose | Loaded by Claude |
|---|---|---|
| `CLAUDE.md` | Working rules: history, tests, manual, permissions, audit, reuse, commits | Every session |
| `ARCHITECTURE.md` | Stack, folder layout, feature naming, dependency rules | Every session (imported) |
| `PROJECT.md` | Features, business rules, entities | When referenced |
| `.claude/rules/*.md` | Short path-triggered reminders | When matching files are opened |
| `.claude/settings.json` | Turns off Claude's commit and PR attribution | Every session (settings) |
| `.gitignore` | Keeps `.env` secrets, `node_modules`, build output and Supabase CLI state out of git | No (git only) |
| `docs/architecture/*.md` | Data layer, permissions, audit trail, migrations, testing | When a rule points there |
| `docs/design/ui.md` | Visual design system | When a rule points there |
| `docs/changelog/<feature>/` | Why things changed, per feature | Before touching that feature |
| `docs/manual/<feature>.md` | End-user manual, per feature | Before touching that feature |

---

## Start a new project

### 1. Create the repository from this template

**On GitHub:** open this repository, click **Use this template › Create a new repository**,
name it, and choose its visibility.

**Or with the GitHub CLI:**

```bash
gh repo create my-app --template maikforte/project-brains --private --clone
```

The new repository gets a copy of these files with a fresh history. It is not a fork,
and later changes to this template don't reach it (see [Pulling template updates](#pulling-template-updates)).

### 2. Clone it

Skip this if you used `--clone` above.

```bash
git clone https://github.com/<you>/my-app.git
```

### 3. Fill the placeholders

Search the whole repository for `{{` and replace every one:

| File | Fill in |
|---|---|
| `CLAUDE.md` | Project name and one-paragraph summary |
| `PROJECT.md` | Overview, users, features, entities |
| `ARCHITECTURE.md` | UAT and prod Supabase projects |
| `docs/architecture/data-layer.md` | Business time zone |
| `docs/design/ui.md` | Character, colour tokens, font, radius, responsive approach |
| `docs/manual/README.md` | Manual page list and anything out of scope |

The `{{...}}` lines in `_template.md` files stay. They are templates for future entries.

### 4. Scaffold the app

Start a Claude session in the new repository and ask:

> Scaffold this project following ARCHITECTURE.md and the README.

That covers:
- a Vite React TypeScript app with Tailwind, shadcn/ui, lucide-react and React Router
- Vitest, React Testing Library, Storybook and PGlite
- `supabase init`, then the folder layout from `ARCHITECTURE.md`
- `package.json` scripts listed in `docs/architecture/testing.md`

### 5. Write the foundation migration

One migration creating `profiles`, `roles`, `permissions`, `role_permissions`,
`has_permission()`, `my_permissions()`, `audit_log`, `audit_row_change()`, `log_event()`
and `assert_table_guards()`.

The SQL is ready to copy from the **Foundation SQL** sections of
`docs/architecture/auth-permissions.md` (first) and `docs/architecture/audit-trail.md`
(second). Then create the super admin by hand, as described in `auth-permissions.md`.

### 6. Make it your project

1. **Replace this README** with one describing your project: what it is, how to run it, environments.
2. **List your features** in `PROJECT.md` and `docs/changelog/README.md`.
3. **Write the first history entry:** `docs/changelog/general/YYYY-MM-DD-project-setup.md`.
4. **Commit:** `chore: scaffold project from project-brains template`.

---

## Pulling template updates

Repositories created from a template don't follow it. To bring a later template change
into a project:

```bash
git remote add template https://github.com/maikforte/project-brains.git
git fetch template
git log template/main --oneline
git cherry-pick <commit>
```

Expect conflicts in files the project has filled in (`CLAUDE.md`, `PROJECT.md`,
`docs/design/ui.md`). Keep the project's content and take only the rule change. Record
it in `docs/changelog/general/`.

---

## Maintaining this template

- **Keep it generic.** A lesson every project should know belongs here. Project-specific
  rules (brand colours, client business rules, print layouts) stay in that project.
- **Make each rule change its own commit**, so projects can cherry-pick it on its own.
- **Commits follow Conventional Commits** (CLAUDE.md rule 12), with no AI attribution.
- **Keep the foundation SQL runnable.** If you change it, run it in PGlite or a local
  Supabase before committing.
- **Template repository setting:** GitHub › **Settings › General › Template repository**
  must stay checked for **Use this template** to appear.
