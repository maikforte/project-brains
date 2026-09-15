# {{PROJECT_NAME}}

> What the app does. How it is built is in `ARCHITECTURE.md`; why it changed is in `docs/changelog/`.

## Overview

{{Two or three paragraphs: the problem, who has it, and what this app does about it.}}

### Client / deployment

{{Who this is for, where it runs, anything unusual about their operation.}}

---

## Users

Roles are **data**, created in the app by the super admin as the business needs them
(see `docs/architecture/auth-permissions.md`). This table describes the *kinds* of
people who use the app, not a fixed role list.

| Kind of user | What they need to do |
|---|---|
| Super admin | Set up roles and permissions, manage users, read the audit log |
| {{User type}} | {{Their main jobs}} |

---

## Features

Each feature's **name** is used for its code folder, changelog folder, manual page
and permission codes (see ARCHITECTURE.md → Feature naming). Add a feature here
**before** writing its code.

| Name | Title | Status | Permissions |
|---|---|---|---|
| `general` | Cross-cutting work (not a real feature) | — | — |
| `users` | User management | Planned | `users.view`, `users.edit` |
| `roles` | Roles & permissions | Planned | `roles.view`, `roles.edit` |
| `audit-log` | Audit log screen | Planned | `audit.view` |
| `{{feature-name}}` | {{Title}} | Planned | `{{feature}}.view`, `{{feature}}.edit` |

Status: Planned → In progress → Shipped.

### Feature: `{{feature-name}}`

**User story:** As a {{user}}, I want to {{action}} so that {{outcome}}.

**Scope:**
- {{What is in}}

**Out of scope:**
- {{What is deliberately not in, so nobody adds it assuming it was forgotten}}

**Key entities:** `{{Entity}}`

**Statuses:** `{{Draft}}` → `{{Active}}` → `{{Closed}}`

**Business rules:**
- {{Rule}}

---

## Business rules (cross-feature)

- {{Rules that apply across features, e.g. "records are never hard-deleted once referenced"}}

---

## Entities

| Entity | Table | Description |
|---|---|---|
| Profile | `profiles` | App user, linked to `auth.users` |
| Role | `roles` | Named set of permissions, created in the app |
| Permission | `permissions` | One permission code, added by each feature's migration |
| Audit entry | `audit_log` | One recorded data change or event |
| {{Entity}} | `{{table}}` | {{Description}} |

---

## Status reference

| Entity | Status | Meaning | Badge variant |
|---|---|---|---|
| {{Entity}} | `{{Status}}` | {{Meaning}} | `neutral` / `info` / `warning` / `success` / `error` |
