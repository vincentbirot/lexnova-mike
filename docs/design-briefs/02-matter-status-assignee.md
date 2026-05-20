# Brief 02 — Matter Status and Assignee

## Context

Lawyers do not just open matters and chat with documents. They open matters, work on them, hand them off, wait for client responses, send invoices, and close them. A matter has a lifecycle. Today's Lexnova AI has no concept of matter lifecycle: every matter is just "exists" or "deleted". This is the foundational gap that makes Mike feel like a smart document workspace rather than a real practice tool.

This brief introduces two fields per matter: a status (an enum representing where the matter is in its lifecycle) and an assignee (the lead lawyer responsible). Both are visible everywhere matters appear — in the matter list, in the matter header, in the sidebar. Both are filterable. They underpin every later feature: tasks (assigned by status), time clock (logged against active matters), Gmail integration (matters in "awaiting client" surface differently), partner dashboard (filter by status across the firm).

## Audience

Every Lex Nova lawyer; especially relevant for partners managing many matters across associates.

## Primary user stories

A partner opens the matter list and filters to "Awaiting client > 7 days". She sees four matters that are stuck. She clicks into the first one to chase the client.

An associate opens his "My matters" view. He sees only matters where he is the primary assignee. Six matters, color-coded by status. He clicks the active ones first.

A senior partner re-assigns a matter from one associate to another. He clicks the assignee chip on the matter, picks a new assignee from a dropdown of firm members, the change saves immediately, the previous assignee gets a toast notification on their next page load.

A junior associate marks a matter as "In review" when she finishes a draft. The partner who is the assignee gets surfaced this matter at the top of her dashboard with a "ready to review" indicator.

A matter is "Billed" — work is done, invoice sent, awaiting payment. The matter still appears in lists but with a muted appearance. Once payment is received, it moves to "Closed", and is hidden from default views unless explicitly searched.

## Schema delta (UI-facing only)

- `projects.status` — enum text: `active`, `awaiting_client`, `in_review`, `billed`, `closed`. Default `active` on creation.
- `projects.assignee_user_id` — uuid foreign key to `auth.users(id)`. Default the creator's user id on creation.

The existing `shared_with` jsonb array remains as the list of additional collaborators. Assignee is the single owner; shared_with is the team.

## Screens

### Matter list (`/projects` aka Matters)

**Update to existing list:**

Two new columns inserted between Name and CM number: Status (chip), Assignee (avatar + name, truncated). Total visible columns become: checkbox, name, status, assignee, CM, files, chats, tabular reviews, created.

Status chip is a small rounded-full badge with semantic color and label:

- `active` — neutral gray background, foreground gray, label "Active"
- `awaiting_client` — amber `oklch(0.828 0.189 84.429)` background tint, label "Awaiting client"
- `in_review` — soft blue `--color-blue-100` background, label "In review"
- `billed` — purple-pink-ish using existing chart color, label "Billed"
- `closed` — fully muted gray, foreground `text-muted-foreground`, label "Closed"

Assignee column shows a small (24px) avatar circle with the user's first initial, color from a deterministic hash of their user id (so each lawyer has a stable color), and the name truncated at 16 chars with ellipsis. Hovering shows the full name and email in a tooltip. Clicking the cell opens an inline reassignment popover (small dropdown of firm users + search).

**New filter tabs above the list:**

The existing tab strip "All | Mine | Shared with me" gains a fourth tab: "Active" (active + awaiting_client + in_review only, hides billed and closed by default). New default tab is "Active". Switch to "All" to see closed too.

Below the tab strip, a row of filter pills: Status (multi-select dropdown), Assignee (multi-select dropdown of firm members). Selected filters appear as chips with X to remove.

### Matter header (`/projects/[id]`)

The current matter detail page header (matter name + breadcrumb + actions) gains:

- A status chip immediately to the right of the matter name, clickable to change status (popover with the 5 status options as radio buttons, save on selection)
- An assignee chip immediately right of the status, showing avatar + name, clickable to reassign (same popover as in list)
- The right-side actions ("+ Chat", "+ Tabular Review") remain unchanged

### Sidebar (recent chats grouped by status)

In the sidebar's "Recent chats" section, each chat shows the matter name and a tiny status dot (4px circle, same color as the status chip) to the left of the chat name. Lawyers can scan the sidebar and immediately see which matters are live versus closed.

### Empty states

- "Active" tab with no active matters: "No active matters. Create one to get started." with primary CTA "+ New matter"
- "Mine" tab with no assigned matters: "Nothing assigned to you. The partners will assign matters as they come in." (no CTA)

## Component additions

- `<StatusChip status={...}>` — semantic color + label
- `<AssigneePill userId={...}>` — avatar + name, hover tooltip
- `<StatusPopover>` — radio-list popover for changing status
- `<AssigneePopover>` — searchable dropdown of firm members for reassignment
- `<MatterFilterBar>` — the row of status + assignee multi-selects with selected chips

Existing components updated:

- `<ProjectsOverview>` table: add the two new columns
- Matter detail header: add the chips inline with the title
- `<AppSidebar>` recent chats section: add the status dot

## Edge cases

**Reassigning to yourself**: no-op, but toast confirms "You are now the assignee."

**Reassigning a matter you're not on**: the receiving lawyer gets a notification (in-app toast on their next page load + future email when Brief 06 lands). The previous assignee remains in `shared_with` so they retain access.

**Closing a matter with active in-flight requests**: prompt "This matter has 2 requests still processing. Close anyway?" — closing cancels the requests.

**Assignee leaves the firm**: if their auth user is deleted, the matter's assignee_user_id becomes null. UI shows a warning chip "Unassigned" in red. Partner-side admin must reassign.

**Status of `closed` and document uploads**: should we allow uploads to closed matters? Yes, for archival completeness. But warn the user with a confirm: "This matter is closed. Add document anyway?"

## Visual references

The existing `ProjectsOverview` table is the structural reference for the list. The status chip pattern matches GitHub's PR status badges in feel — small, semantic-colored, immediately readable. The assignee avatar pattern matches Linear's issue assignee — small circular initial, color-hashed, tooltip on hover.

## Accessibility

- Status chips have a text label, never color-only
- Assignee avatars have an `aria-label` with the full name
- Status popover and assignee popover are keyboard navigable (Tab to focus, Enter to select, Esc to close)
- Filter dropdowns announce the count of selected items for screen readers

## Acceptance criteria

Every matter has a status visible in the list and the matter header. Every matter has a single primary assignee visible. Status and assignee are filterable in the matter list. The default matter list shows only active matters; closed and billed are hidden until the user switches tabs or filters. Reassigning a matter updates immediately and the previous assignee retains read access via shared_with. The sidebar shows status dots next to recent matters.

## Out of scope

- Assignee notifications via email or Slack (Brief 06, 07)
- Bulk reassignment (out of scope; tackle if requested after roll-out)
- Custom statuses defined by the firm (later, requires admin panel)
- Status history / audit log (later, requires audit table)
- Auto-status transitions based on activity (e.g., move to "awaiting_client" when no firm activity for 7 days — requires Gmail integration and a background job)
