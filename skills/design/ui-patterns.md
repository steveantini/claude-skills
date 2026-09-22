# UI Patterns

| Field | Value |
|---|---|
| **Version** | 1.1.0 |
| **Last Updated** | 2026-09-22 |
| **Applicability** | React/Next.js applications, component-based UI development |
| **Dependencies** | Tailwind CSS (or CSS-in-JS), Radix UI / shadcn/ui (recommended), Lucide icons |

---

## Design System Approach

### Architecture Layers

```
Tokens       → Colors, spacing, typography, shadows, radii
Primitives   → Unstyled, accessible base components (Radix UI)
Components   → Styled, single-purpose UI elements (Button, Input, Badge)
Compositions → Multi-component patterns (DataTable, FormField, Dialog)
Templates    → Page-level layouts (DashboardLayout, AuthLayout)
```

### Design Tokens

Define tokens as CSS custom properties or Tailwind config. Single source of truth.

```css
/* globals.css — Token layer */
:root {
  /* Colors — semantic naming */
  --color-background: 0 0% 100%;
  --color-foreground: 222 47% 11%;
  --color-primary: 222 47% 11%;
  --color-primary-foreground: 210 40% 98%;
  --color-muted: 210 40% 96%;
  --color-muted-foreground: 215 16% 47%;
  --color-destructive: 0 84% 60%;
  --color-border: 214 32% 91%;

  /* Spacing — consistent scale */
  --spacing-xs: 0.25rem;   /* 4px */
  --spacing-sm: 0.5rem;    /* 8px */
  --spacing-md: 1rem;      /* 16px */
  --spacing-lg: 1.5rem;    /* 24px */
  --spacing-xl: 2rem;      /* 32px */
  --spacing-2xl: 3rem;     /* 48px */

  /* Radii */
  --radius-sm: 0.25rem;
  --radius-md: 0.375rem;
  --radius-lg: 0.5rem;
  --radius-full: 9999px;

  /* Shadows */
  --shadow-sm: 0 1px 2px rgba(0,0,0,0.05);
  --shadow-md: 0 4px 6px -1px rgba(0,0,0,0.1);
  --shadow-lg: 0 10px 15px -3px rgba(0,0,0,0.1);
}
```

### Component API Conventions

- **Variants**: Use `variant` prop for visual styles (`default`, `destructive`, `outline`, `ghost`).
- **Sizes**: Use `size` prop (`sm`, `md`, `lg`).
- **Composition**: Use `asChild` pattern (Radix) for polymorphic rendering.
- **Naming**: PascalCase for components, camelCase for props.
- **Forward refs**: Always forward refs on interactive elements.
- **className merging**: Accept and merge external `className` via `cn()` utility.

```typescript
// lib/utils.ts
import { clsx, type ClassValue } from 'clsx';
import { twMerge } from 'tailwind-merge';

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

---

## Common UI Patterns

### Data Tables

**Structure**:

```
DataTable
  ├── Toolbar (search, filters, bulk actions)
  ├── Table
  │   ├── TableHeader (sortable columns)
  │   └── TableBody
  │       └── TableRow (selectable, clickable)
  ├── EmptyState (when no results)
  └── Pagination (page size, page navigation)
```

**Key behaviors**:
- Column sorting: Click header to toggle asc/desc/none.
- Row selection: Checkbox column, select all, bulk actions appear in toolbar.
- Pagination: Show total count, page size selector (10/25/50), prev/next.
- Loading: Skeleton rows matching expected structure, not spinner.
- Empty: Illustration + message + action ("No projects yet. Create one.").
- Responsive: Horizontal scroll on mobile, or switch to card layout.

**State management**: Keep sort, filter, pagination in URL params for shareable/bookmarkable state.

```typescript
// URL-driven table state
const searchParams = useSearchParams();
const page = Number(searchParams.get('page') ?? '1');
const sort = searchParams.get('sort') ?? 'created_at';
const order = searchParams.get('order') ?? 'desc';
```

### Forms

**Structure**:

```
Form
  └── FormField
      ├── Label
      ├── Input / Select / Textarea / etc.
      ├── Description (optional helper text)
      └── ErrorMessage (validation error)
```

**Key behaviors**:
- Validate on blur (first interaction), then on change (subsequent).
- Show inline errors below the field, not in a summary.
- Disable submit button while submitting; show loading indicator.
- Preserve form state on error (never clear the form).
- Group related fields with `fieldset` and `legend`.
- Use `react-hook-form` + `zod` for validation.

**Accessibility**:
- Associate labels with inputs via `htmlFor`/`id`.
- Use `aria-describedby` for helper text and errors.
- Use `aria-invalid="true"` on invalid fields.
- Set `role="alert"` on error messages.

### Modals / Dialogs

**When to use**:
- Confirmation of destructive actions.
- Quick creation forms (without leaving context).
- Focused tasks requiring attention.

**When NOT to use**:
- Long forms (use a dedicated page).
- Information display (use inline expansion or a panel).
- Nested modals (never).

**Key behaviors**:
- Focus trap: Tab cycles within the dialog.
- Escape to close.
- Click overlay to close (unless destructive confirmation).
- Return focus to trigger element on close.
- Animate in/out (scale + fade, 150ms).
- Scroll within dialog body if content overflows; header/footer fixed.

**Sizes**: `sm` (400px), `md` (500px), `lg` (640px), `xl` (780px), `full` (90vw).

### Toasts / Notifications

**Types**: `success`, `error`, `warning`, `info`.

**Key behaviors**:
- Auto-dismiss: 5s for success/info, persistent for errors.
- Stack from bottom-right (or top-right); max 3 visible.
- Include dismiss button.
- Errors: include action ("Retry" button).
- Never use toasts for critical errors that block the user.

**Content pattern**:
- Success: "Project created" (past tense, no "successfully").
- Error: "Could not save changes. Please try again." (what happened + what to do).
- Warning: "You have unsaved changes." (what to be aware of).

### Loading States

**Hierarchy**:

| State | Pattern | When |
|---|---|---|
| Initial page load | Full skeleton layout | First load, no cached data |
| Section loading | Skeleton for that section only | Partial data refresh |
| Action pending | Button spinner + disabled state | Form submit, API call |
| Background refresh | Subtle indicator (opacity change, small spinner) | Revalidation, polling |
| Navigation | Progress bar (top of page) | Route change |

**Skeleton guidelines**:
- Match the shape of the actual content.
- Use pulsing animation (not spinning).
- Show 3-5 skeleton rows for lists.
- Never show skeleton for more than 3 seconds; show error/retry after timeout.

---

## Responsive Layout

### Layout Primitives

```typescript
// Stack — vertical spacing
<div className="flex flex-col gap-4">

// Row — horizontal with wrap
<div className="flex flex-wrap gap-4">

// Grid — responsive columns
<div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">

// Sidebar layout
<div className="flex min-h-screen">
  <aside className="hidden lg:block w-64 border-r" />
  <main className="flex-1 p-6" />
</div>

// Container — max width + centered
<div className="mx-auto max-w-7xl px-4 sm:px-6 lg:px-8">
```

### Page Layout Patterns

| Layout | Use Case |
|---|---|
| Single column | Content pages, settings, forms |
| Sidebar + content | Dashboards, admin panels, docs |
| Two column (equal) | Comparison views |
| Grid | Cards, gallery, product listings |
| Split (50/50 or 40/60) | Auth pages, landing sections |

---

## Dark Mode

### Implementation

Use CSS custom properties switched by a class on `<html>`.

```css
.dark {
  --color-background: 222 47% 11%;
  --color-foreground: 210 40% 98%;
  --color-primary: 210 40% 98%;
  --color-primary-foreground: 222 47% 11%;
  --color-muted: 217 33% 17%;
  --color-muted-foreground: 215 20% 65%;
  --color-border: 217 33% 17%;
}
```

### Guidelines

- **Never use pure black** (`#000`). Use dark gray (`hsl(222, 47%, 11%)`).
- **Reduce contrast slightly**: Text at 87-90% brightness, not 100%.
- **Elevate with lightness**: Higher surfaces are lighter (opposite of light mode shadows).
- **Desaturate colors**: Vibrant colors strain eyes on dark backgrounds.
- **Test both modes**: Every component, every state.
- **Persist preference**: `localStorage` + `prefers-color-scheme` media query.
- **Prevent flash**: Set theme in `<head>` before body renders (blocking script).

### Theme Toggle

```typescript
function useTheme() {
  const [theme, setTheme] = useState<'light' | 'dark' | 'system'>('system');

  useEffect(() => {
    const root = document.documentElement;
    const systemDark = window.matchMedia('(prefers-color-scheme: dark)').matches;
    const isDark = theme === 'dark' || (theme === 'system' && systemDark);
    root.classList.toggle('dark', isDark);
  }, [theme]);

  return { theme, setTheme };
}
```

---

## Animation Principles

### Duration Scale

| Category | Duration | Use |
|---|---|---|
| Instant | 0ms | Focus rings, color changes |
| Fast | 100-150ms | Hover states, button feedback, tooltips |
| Normal | 200-300ms | Modals, dropdowns, panels |
| Slow | 300-500ms | Page transitions, large layout shifts |

### Easing

- **Enter**: `ease-out` (fast start, slow finish — feels responsive).
- **Exit**: `ease-in` (slow start, fast finish — feels natural).
- **Move**: `ease-in-out` (smooth repositioning).
- **Spring**: For playful, physical-feeling interactions.

### What to Animate

- **Do**: Opacity, transform (translate, scale), height/max-height for expand.
- **Avoid**: Layout properties (width, padding, margin) — use transform instead.
- **Never animate**: Properties that trigger layout thrashing on every frame.

### Reduce Motion

Always respect user preference:

```css
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
  }
}
```

---

## Icon System

### Approach

Use a single icon library consistently. Recommended: **Lucide** (open source, tree-shakable, consistent style).

### Conventions

- **Size**: Match text — 16px (inline), 20px (buttons), 24px (standalone).
- **Color**: `currentColor` (inherits text color).
- **Stroke width**: 2px default; 1.5px for detailed icons at small sizes.
- **Spacing**: 8px gap between icon and label.
- **Accessibility**: Decorative icons get `aria-hidden="true"`. Meaningful icons get `aria-label`.

```tsx
import { Plus, Trash2, Settings } from 'lucide-react';

// Decorative (label visible)
<Button><Plus className="h-4 w-4 mr-2" aria-hidden="true" />Create</Button>

// Standalone (no visible label)
<Button variant="ghost" size="icon" aria-label="Delete">
  <Trash2 className="h-4 w-4" />
</Button>
```

---

## Typography Scale

### Scale (Tailwind)

| Level | Class | Size | Weight | Use |
|---|---|---|---|---|
| Display | `text-4xl` | 36px | 700 | Hero headings |
| H1 | `text-3xl` | 30px | 700 | Page titles |
| H2 | `text-2xl` | 24px | 600 | Section headings |
| H3 | `text-xl` | 20px | 600 | Sub-sections |
| H4 | `text-lg` | 18px | 600 | Card titles |
| Body | `text-base` | 16px | 400 | Default text |
| Body small | `text-sm` | 14px | 400 | Secondary text, table cells |
| Caption | `text-xs` | 12px | 400 | Labels, timestamps, badges |

### Typography Rules

- **Line height**: 1.5 for body, 1.2 for headings.
- **Max line width**: 65-75 characters for readability (`max-w-prose`).
- **Font stack**: System fonts for performance, or one custom font maximum.
- **Hierarchy**: Use weight and size, not just size. Bold heading + regular body creates clear contrast.
- **Monospace**: Use for code, IDs, technical values. `font-mono`.

### System Font Stack

```css
font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto,
  'Helvetica Neue', Arial, sans-serif;
```

---

## Component Checklist — New Component

- [ ] Accepts and merges `className` prop
- [ ] Forwards `ref`
- [ ] Has defined variants and sizes via `cva` or equivalent
- [ ] Keyboard accessible (Enter/Space for buttons, Escape for dismissals)
- [ ] Focus visible styles (ring)
- [ ] Dark mode tested
- [ ] Loading / disabled states defined
- [ ] Error states defined (for inputs)
- [ ] Responsive behavior defined
- [ ] Motion respects `prefers-reduced-motion`
- [ ] Documented with example usage

## One Source for a Rendered Label

**The rule.** A user-visible label that appears in more than one place (a
navigation rail, a breadcrumb, a page title and browser tab, marketing
mock-ups of the product) lives in one exported constant, and every surface
reads it. Identifiers keep their original word when the label changes:
routes, storage and preference keys, query parameters, group keys, table and
column names, and code names are never renamed for a label.

**Why.** A label typed in six files drifts in six directions, and the
marketing mock-up is the first to show a word the product no longer uses.
Identifiers are contracts with stored data, bookmarks, and other code; a
rename there is a migration with no user benefit, so the label and the
identifier are allowed to disagree on purpose, and the constant's comment
says so.

**Example.**

```ts
// lib/labels.ts
/** The label users see for the group. The identifier keeps the old word by
 *  rule: the route, the table, and the stored keys still say "sections". */
export const SECTIONS_LABEL = "Topics";
```

```tsx
<RailGroup caption={SECTIONS_LABEL} captionHref="/workspace/sections" groupKey="sections" />
// breadcrumb segments: [HOME, SECTIONS_LABEL, item.name]
// page: export const metadata = { title: SECTIONS_LABEL };
// the marketing mock's rail caption: <NavCaption>{SECTIONS_LABEL}</NavCaption>
```

**How to test for it.** One test renders each surface and asserts it reads
the constant, and a source scan asserts the old label appears nowhere as a
standalone string:

```ts
expect(renderToStaticMarkup(<Breadcrumb .../>)).toContain(`>${SECTIONS_LABEL}<`);
expect(readFileSync("app/workspace/sections/page.tsx", "utf8")).toContain("title: SECTIONS_LABEL");
// no file in app/ or components/ carries the retired label as a label
expect(offenders(/"Sections"|>\s*Sections\s*</)).toEqual([]);
// the identifiers are untouched
expect(railSource).toContain('groupKey="sections"');
```

## The Product Name: One Value, One Component, Scanned Casing

**The rule.** The product name is one config value, rendered through one
component that pins `text-transform: none` inline, and read as a string
through one exported constant everywhere a string is needed (titles, prompts,
export footers). It is never typed as a literal, and it never sits inside an
element that uppercases its text.

**Why.** A name with internal casing (a lowercase first letter, a capital
inside) is flattened by any uppercased container, title-cased by any
humanizing helper, and mistyped by hand. A scan for the wrong spellings is
cheap; a review is not, and it misses the case where the uppercasing comes
from a helper that composes class names, which a scan for a literal
`className="... uppercase ..."` never sees.

**Example.**

```tsx
export function Wordmark({ className }: { className?: string }) {
  return <span className={className} style={{ textTransform: "none" }}>{siteConfig.name}</span>;
}
export const PRODUCT_NAME = siteConfig.name;

// In an uppercased eyebrow: the mark, never the string.
<p className="font-mono uppercase">Powered by <Wordmark /></p>
```

**How to test for it.**

```ts
// The wrong spellings, in every rendered surface and every source file.
for (const wrong of ["Nameos", "NAMEOS", "NameOS"]) expect(text).not.toContain(wrong);
// No code re-cases the name.
expect(source).not.toMatch(/PRODUCT_NAME\s*\.\s*(toUpperCase|toLowerCase|charAt|replace)\b/);
// The bare string never sits inside an uppercasing element, including one whose
// classes come through a helper (cn, clsx, cva): scan the helper's arguments too.
const uppercasing = /<(\w+)[^>]*(?:className="[^"]*\buppercase\b[^"]*"|className=\{cn\([^)]*\buppercase\b[^)]*\))[^>]*>([\s\S]*?)<\/\1>/g;
for (const m of source.matchAll(uppercasing)) expect(m[2]).not.toMatch(/\{PRODUCT_NAME\}/);
```

Name the components that uppercase through a helper (a mono-caps label
component, for instance) in the test's comment, so the next writer knows the
literal scan is not enough.
