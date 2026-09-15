# UI design system

How the app looks and how components are built. Fill every `{{placeholder}}` when the
project starts. After that, changes go through CLAUDE.md rule 6.

---

## Character

{{One line on how the app should feel to use.}}

**Aim for:** {{Three or four qualities the design must have}}

**Avoid:** {{Styles or patterns that would feel wrong for this app}}

---

## Colour tokens

**Use semantic tokens only.** Never raw Tailwind colours (`bg-blue-500`) or hex values
in components. Define tokens as CSS variables in `src/index.css` and map them in
`tailwind.config`.

| Token | Value | Use |
|---|---|---|
| `--primary` | {{#hex}} | Primary actions, key UI elements |
| `--primary-hover` | {{#hex}} | Hover on primary |
| `--accent` | {{#hex}} | Links, active states, focus |
| `--background` | {{#hex}} | Page background |
| `--surface` | {{#hex}} | Cards, panels, modals |
| `--surface-alt` | {{#hex}} | Alternating rows, subtle sections |
| `--border` | {{#hex}} | Borders, dividers |
| `--text-primary` | {{#hex}} | Main text |
| `--text-secondary` | {{#hex}} | Labels, secondary text |
| `--text-muted` | {{#hex}} | Placeholders, disabled |

### Status tokens

| Token | Value | Use |
|---|---|---|
| `--success` / `--success-bg` | {{#hex}} / {{#hex}} | Completed, active, approved |
| `--warning` / `--warning-bg` | {{#hex}} / {{#hex}} | Pending, low, needs attention |
| `--error` / `--error-bg` | {{#hex}} / {{#hex}} | Failed, blocked, destructive |
| `--info` / `--info-bg` | {{#hex}} / {{#hex}} | In progress, informational |

Status colours mean the same thing on every screen. Map each entity status to a variant
in the `PROJECT.md` status reference.

**Dark mode:** {{Supported / Not supported}}. If supported, redefine the same tokens
under `.dark`. Components don't change.

---

## Typography

- Font: {{e.g. Inter, system-ui stack}}. Monospace: {{e.g. JetBrains Mono}}, for codes,
  IDs, reference numbers and aligned figures.
- Weights: 400 body, 500 labels and emphasis, 600–700 headings.
- Minimum 14px for body and table text.

| Class | Size | Use |
|---|---|---|
| `text-xs` | 12px | Metadata, timestamps |
| `text-sm` | 14px | Body, table cells, form fields |
| `text-base` | 16px | Primary content |
| `text-lg` | 18px | Section headers |
| `text-xl` | 20px | Page section titles |
| `text-2xl` | 24px | Page headers |

---

## Spacing, shape, motion

- **8px grid.** Page padding 24px desktop / 16px mobile. Card padding 20–24px. Section gaps 24–32px.
- **Radius** {{e.g. 6–8px}}. Cards use a border **or** a shadow, not both.
- **Motion is functional only:** hover 150ms, panel/modal/dropdown 150–200ms, skeleton
  pulse. Nothing over 300ms. No bounce, parallax or staggered list animations.
- **Responsive:** {{Desktop-first / Mobile-first}}. Tables scroll horizontally or become
  cards on mobile. Critical actions are always reachable.

---

## Components

### Where they live

| Kind | Location |
|---|---|
| shadcn/ui generated primitives | `src/components/ui/primitives/`, set as the `ui` alias in `components.json` |
| Project shared components | `src/components/ui/<Name>/` |
| Used by one feature only | `src/features/<feature>/` |

Project components wrap or compose shadcn primitives. Keep edits to the generated
primitives minimal so they stay easy to update, and put project styling in the wrapper.

### Reuse before creating (CLAUDE.md rule 10)

1. **Search** `src/components/ui/`, `src/lib/`, and other features' `index.ts`.
2. **Extend** with a variant, size or prop if something is close.
3. **Compose** existing components if the need is a combination of them.
4. **Create** only if nothing fits, and note what was checked in the changelog entry.
5. **Promote** to `src/components/ui/` when a second feature needs it.

Signs you should extend instead of create: the new component would copy more than a
few lines of an existing one, or differ only in colour, size or icon.

### Required files

```
src/components/ui/StatusBadge/
├── StatusBadge.tsx          # implementation, props interface exported
├── StatusBadge.test.tsx     # see docs/architecture/testing.md
├── StatusBadge.stories.tsx  # every variant + loading/empty/error
└── index.ts                 # export { StatusBadge } from './StatusBadge'; export type { StatusBadgeProps } ...
```

### Variants with `cva`

```tsx
import { cva, type VariantProps } from 'class-variance-authority';
import { cn } from '@/lib/utils';

const statusBadgeVariants = cva(
  'inline-flex items-center gap-1 rounded-md px-2.5 py-0.5 text-sm font-medium',
  {
    variants: {
      status: {
        neutral: 'bg-surface-alt text-text-secondary',
        success: 'bg-success-bg text-success',
        warning: 'bg-warning-bg text-warning',
        error: 'bg-error-bg text-error',
        info: 'bg-info-bg text-info',
      },
    },
    defaultVariants: { status: 'neutral' },
  },
);

export interface StatusBadgeProps
  extends React.HTMLAttributes<HTMLSpanElement>,
    VariantProps<typeof statusBadgeVariants> {}

export function StatusBadge({ status, className, ...props }: StatusBadgeProps) {
  return <span className={cn(statusBadgeVariants({ status }), className)} {...props} />;
}
```

A new look is a new variant, not a new component.

---

## Patterns

- **Lists:** a shared `DataTable`, paged on the server, with sticky header, sortable
  columns, row hover, and loading skeleton, empty state and error state.
- **Forms:** a shared `FormField` (label + input + inline error). Validation errors show
  inline, never in a modal. Required fields are marked.
- **Actions:** primary action top-right. Secondary actions in a menu. **Destructive
  actions need a confirm dialog** that names what will be affected.
- **Feedback:** a toast after every save, delete, export or import, naming what happened.
- **Permission-gated controls** are hidden (not just disabled) when `can()` is false,
  unless the user needs to know the action exists.
- **Codes, IDs and aligned numbers** use the monospace font.
- **Icons:** lucide-react only.

---

## Accessibility

Required, not optional.

- **Colour is never the only signal.** A status always has text or an icon as well.
- Every input has a label. Icon-only buttons have `aria-label`.
- Visible focus rings on everything interactive. Full keyboard use, including dialogs
  (focus trapped, Esc closes).
- Text contrast meets WCAG AA.
- Storybook's a11y addon runs on every story. Fix violations, don't disable the check.
