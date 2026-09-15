# User manual

Written for **end users**, the people using the app, not developers. One file per
feature: `docs/manual/<feature>.md`, named the same as the feature folder.

The rules for when to update are CLAUDE.md rule 5. The short version: if a user would
do something differently after the change, the manual changes in the same commit.

---

## Pages

Keep this list in the order a new user would learn the app.

| Page | Feature |
|---|---|
| {{[Getting started](getting-started.md)}} | {{Navigation, signing in, common screen patterns}} |
| {{[Feature title](feature-name.md)}} | {{One line}} |

---

## Style

- **Start from `_template.md`.**
- **Write for the task, not the screen.** "How to submit an expense claim", not "The Expenses page".
- **Lead with what goes wrong.** A manual that only lists fields is a worse copy of the
  UI. Say what trips people up, what a warning means, and how to recover.
- **Numbered steps** for procedures. One action per step.
- **Use the exact labels the UI shows,** in bold: click **Save**.
- **Show where things are** as a path: **Settings › Users › New user**.
- **Link to other pages by file,** not by section number: `see [Roles](roles.md#creating-a-role)`.
  Numbered cross-references break silently when chapters move.
- **Say who can do it.** If an action needs a permission, say so in words ("needs the
  *Approve orders* permission"), not the code.
- No screenshots unless a layout is genuinely hard to describe. They go stale first.
- Plain language. Short sentences. Explain domain terms the first time they appear.

## Not in the manual

- Developer details: tables, APIs, permission codes
- Refactors, performance work, styling that doesn't change what users do
- {{Anything the project deliberately leaves out, e.g. administration screens}}
