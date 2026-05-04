# Lexnova AI — Design System

This document captures the existing visual language inherited from the Mike open-source baseline. All new feature designs MUST extend this system rather than introduce new patterns. The goal is internal coherence: a lawyer should not be able to tell which screens existed in the upstream and which were added for Lex Nova.

## 1. Foundation

### Stack

- Tailwind CSS v4 with CSS custom properties for theming
- shadcn/ui components, "new-york" style variant
- Lucide React icons (no other icon library)
- Inter for body and UI, EB Garamond for serif/legal-document text

### Theme strategy

Light theme is default. Dark mode is supported via the `.dark` class on the root, all colors switch through the same tokens. New screens must work in both modes. Colors are expressed in `oklch()` to give the palette a perceptually uniform feel.

## 2. Color tokens

All colors live in `frontend/src/app/globals.css` under `:root` and `.dark`. Use the semantic Tailwind tokens (`bg-background`, `text-foreground`, `bg-muted`, etc.), never raw hex.

Light mode primary tokens:

- `--background`: `oklch(1 0 0)` — pure white
- `--foreground`: `oklch(0.145 0 0)` — near black
- `--primary`: `oklch(0.205 0 0)` — black, used for primary buttons and CTAs
- `--primary-foreground`: `oklch(0.985 0 0)` — near white, text on primary
- `--muted`: `oklch(0.97 0 0)` — very pale gray, used for subtle backgrounds and disabled states
- `--muted-foreground`: `oklch(0.556 0 0)` — medium gray, secondary text
- `--border`: `oklch(0.922 0 0)` — light gray, all dividers and form borders
- `--accent`: `oklch(0.97 0 0)` — same as muted, used for hover backgrounds on interactive items
- `--destructive`: `oklch(0.577 0.245 27.325)` — red, for delete actions and errors
- `--ring`: `oklch(0.708 0 0)` — focus ring

Brand accent (used for links, highlights, and active states):

- Azure blue: `rgb(0, 136, 255)` — exposed as `--color-blue` and used as `text-blue` / `bg-blue-50`
- The five `--color-blue-*` shades cascade from 50 (5% opacity) to 700 (saturated)

Sidebar uses its own color tokens (`--sidebar`, `--sidebar-foreground`, `--sidebar-accent`, etc.), all currently mapped to off-whites in light mode and dark grays in dark mode. Treat the sidebar as visually distinct from the main content area.

PDF and DOCX viewers use specific highlight colors that should NOT be reused for general UI:

- PDF text highlight: `rgba(37, 99, 235, 0.2)` (semi-transparent blue)
- DOCX inserted text: green `#16a34a` with `rgba(22, 163, 74, 0.08)` background
- DOCX deleted text: red `#dc2626` with `rgba(220, 38, 38, 0.08)` background and strikethrough

## 3. Typography

Two type families:

**Inter (sans-serif, default body and UI)**

- Page headings, navigation labels, table headers, button text, all interactive copy
- Default weight 400, semibold (600) for headings, regular for body
- Numerals: tabular for tables, regular elsewhere

**EB Garamond (serif, legal text and structured content)**

- Used inside the Tiptap workflow editor (`workflow-editor-content` class)
- Used for `.usc-section` and `.cfr-section` legal-document content
- Conveys authority and continuity with traditional legal publishing
- Weights available: 400, 500, 600, 700

When in doubt, use Inter. Switch to EB Garamond only for content that mimics a legal document being read by a lawyer (extracted text, memo body, citation passages). Never for navigation or chrome.

Type scale (used informally, not codified):

- Page heading: serif EB Garamond, ~24-28px (used in main page titles like "Matters")
- Section heading: sans Inter, semibold, 16px
- Body: sans Inter, 14px
- Caption / table cell: sans Inter, 13-14px
- Help text and timestamps: sans Inter, 12-13px, `text-muted-foreground`

## 4. Layout

### Application shell

Two-column layout enforced by `frontend/src/app/(pages)/layout.tsx`:

- Left: collapsible `AppSidebar` with logo, primary nav, recent chats, user dropdown
- Right: main content area, takes remaining width, has its own scroll context
- Sidebar collapse state persists across reloads (stored in `localStorage` as `sidebarOpen`)

The sidebar contains:

- Top: Mike logo (to be replaced by Lexnova AI mark) and a `PanelLeft` collapse toggle
- Primary nav (always-visible): Assistant, Matters, Tabular Review, Workflows
- Secondary section "Chats" — collapsible list of recent chats with the current matter
- Bottom: user avatar with email, plan tier, and a `ChevronsUpDown` menu opening profile / sign out

### Page structure pattern

Every main content page follows the same skeleton:

```
[Page header — sticky top]
  [Page title in EB Garamond serif]   [Search icon] [Action button(s)]
  [Tab strip: All | Mine | Shared with me] (where applicable)

[Content area — scrollable]
  [Optional table column headers, sticky]
  [List / table / grid of items]
  [Empty state if no items: centered icon + heading + helper text + CTA]
```

Page titles are serif (EB Garamond), all other UI is sans (Inter). Action buttons in headers go to the right (`+` for create, search icon, share icon, etc.).

### Empty states

Centered, vertically and horizontally, in the content area. Pattern:

- Lucide icon (filled, neutral gray, 24-32px)
- Bold heading (serif, e.g., "Matters")
- One- or two-line helper sentence in `text-muted-foreground`
- Single primary CTA button with `+` icon prefix

## 5. Components

### Buttons

shadcn `Button` component, four variants in active use:

- **Primary**: `bg-primary text-primary-foreground` (black with white text), used for the single dominant CTA per surface (e.g., "Create New", "Sign up", "Confirm")
- **Outline**: transparent with `border-border`, used for secondary actions in modals and headers
- **Ghost**: transparent, no border, hover background `bg-accent`, used in toolbars, row actions, sidebar nav
- **Destructive**: `bg-destructive text-destructive-foreground` (red), used in confirm modals for delete actions

Sizes: default (h-9), sm (h-8), icon (h-9 w-9). Almost all surfaces use default. Icon buttons appear in headers and row action menus.

### Tables

The pattern in `ProjectsOverview` is the canonical reference. Every list-of-items page uses it:

- Header row: `border-b border-border`, `bg-background`, sticky if scroll is long
- Columns: name (300px), then metadata columns each ~80-150px, action menu on the right
- First column: 8px-wide checkbox column for bulk selection
- Last column: ellipsis icon (`MoreHorizontal`) for `RowActions` menu
- Hover: `hover:bg-accent/50` on the entire row, cursor-pointer
- Selected: `bg-accent` background, blue check
- Inline rename: double-click on name cell turns it into a small text input with save-on-blur and esc-to-cancel

### Modals (Dialogs)

shadcn `Dialog` component, used for create/edit/share/upload flows. Pattern:

- Centered overlay with `bg-black/50` backdrop
- Card width 480px-560px depending on content density
- Title in serif (EB Garamond), 20px, semibold
- Body content with form fields (label above input, 8px gap)
- Footer right-aligned: outline "Cancel" + primary action button
- Close on backdrop click and Esc

### Form elements

- Inputs: shadcn `Input`, full width within container, h-9, `border-border`, focus ring `--ring`
- Labels: 13px, semibold, 4px below input baseline
- Helper text: 12px, `text-muted-foreground`, below input
- Error state: red border, error message below in `text-destructive`

### Toasts

shadcn `Sonner` toast library. Three variants in use:

- success (default): white card with green check icon, auto-dismiss 4s
- error: red-tinted card with X icon, auto-dismiss 6s
- info / loading: shows spinner during long operations

Toasts appear bottom-right. Maximum 3 stacked. Action buttons supported (e.g., "Undo" on delete).

### Sidebar items

Navigation items: 36px height, 12px padding, 8px gap between icon and label, `rounded-md` on hover. Active state: `bg-sidebar-accent text-sidebar-accent-foreground`. Recent chat items are smaller (32px, `text-sm`), with the chat title truncated with ellipsis.

### Cards (DocumentCard pattern)

Used in matter detail pages. Pattern:

- Border `border-border`, `rounded-lg`, white background
- 16px padding
- Document type icon (top-left, color-coded by file type)
- Filename in 14px medium-weight
- Metadata row (size, date, version) in 12px `text-muted-foreground`
- Hover: subtle border darken and tiny shadow

## 6. Interaction patterns

### Loading

- Page-level loading: skeleton rows or spinner in the content area
- Inline loading: shimmer animation (defined in `globals.css` as `@keyframes shimmer`)
- Submit-pending: button shows spinner replacing icon, button is disabled

### Hover and focus

- All interactive elements have a clear hover state, usually background change to `bg-accent`
- Focus rings use `outline-ring/50` (CSS variable)
- Keyboard navigation should reach every interactive element

### Selection and bulk actions

When items are selected (table rows), a contextual bulk-action bar slides in from the top of the content area, replacing the regular header. It shows count selected and the relevant actions. Esc or clicking deselect-all returns to normal header.

### Inline editing

Used for renaming matters, editing CM numbers, etc. Triggered by double-click. Saves on blur or Enter. Cancels on Esc. Failed saves revert with a toast.

### Confirmation dialogs

For destructive actions (delete matter, delete chat, etc.). Pattern:

- Title: "Delete X?" in serif
- Body: explains consequences, names the resource being deleted
- Footer: outline "Cancel" + destructive "Delete" button
- Never auto-confirm; require explicit click

## 7. Iconography

Lucide React only. Common semantic mappings already in use:

- `MessageSquare` — Assistant / chat
- `FolderOpen` — Matters
- `Table2` — Tabular Review
- `Library` — Workflows
- `User` — profile / account
- `Plus` — create new
- `Search` — header search
- `MoreHorizontal` — row actions menu
- `ChevronDown` / `ChevronsUpDown` — expand/collapse, dropdown affordance
- `PanelLeft` — sidebar toggle
- `Trash2` — delete
- `Pencil` — edit
- `Share` / `Share2` — share with team
- `Download` — export
- `Upload` — upload document
- `FileText` / `File` — generic document
- `FileImage` — image
- `CheckCircle2` — success state
- `AlertCircle` — error state

When introducing a new feature, reuse these where possible. New icons should be Lucide and chosen for clarity over novelty.

## 8. Responsive behavior

The app is desktop-first (lawyers use laptops and desktops). Minimum supported width: 1024px. Tablet (768px-1023px): sidebar auto-collapses to icons only. Mobile (<768px): sidebar becomes a slide-over from the left, content area takes full width, tables condense to card layouts where possible. Mobile is supported but secondary; the canonical experience is desktop.

## 9. Accessibility

- All interactive elements: keyboard reachable, focus visible
- Color contrast: minimum WCAG AA (4.5:1 for body text, 3:1 for large text)
- Icons paired with semantic labels in `aria-label` when label is otherwise visual
- Form inputs: every input has a `<label>` (visible or sr-only)
- Dynamic content updates: announce via `role="status"` for non-urgent, `role="alert"` for errors
- Modal dialogs trap focus, close on Esc, return focus to trigger element on close

## 10. What new feature designs must respect

When proposing a new screen or component:

- Use existing semantic tokens (no new color literals)
- Reuse Lucide icons; don't introduce a second icon library
- Match the page header pattern (serif title + right-aligned actions)
- Match the table pattern for any list of items
- Use shadcn Dialog for any create/edit flow that opens from a button
- Use Sonner toasts for all feedback (success/error/info)
- Default to ghost or outline buttons; reserve primary for the dominant action
- Test in both light and dark modes; both must work
- Preserve EB Garamond serif for any page title or extracted legal-document text
- Do not introduce new layout patterns; new pages live inside the existing two-column shell
