# Brief 05 — DOCX Collaboration (Light)

## Context

Lex Nova drafts contracts and memos in Word. Today, partners send drafts to associates, associates make edits in Word, partners merge or accept/reject — the whole loop happens via email attachments and Word's track changes. It is slow and error-prone.

Lexnova AI has the core of a better solution already in its schema: `document_versions` and `document_edits` tables with `accepted` / `rejected` / `pending` status, plus inline DOCX rendering with insert/delete styling in `globals.css`. What's missing is the multi-user UX: an associate proposes edits, the partner reviews and accepts or rejects each one, the result downloads as a clean DOCX.

This is the "light" version: no real-time co-editing, no live cursors. The "heavy" version (full Google Docs-style collaboration) is a separate workstream estimated at 4-6 weeks. Light version is 1-2 weeks and covers 90% of legal use cases.

## Audience

Every Lex Nova lawyer who drafts or reviews contracts and memos. The most active users are associates (who draft) and partners (who review).

## Primary user stories

A partner uploads a Word draft of an SPA into the Acme matter. The associate opens the matter, sees the draft, opens it in Lexnova AI's DOCX viewer, and clicks "Suggest edits". She is now in proposal mode: she can highlight text and replace it, delete it, or add new text. Each edit saves as a `pending` proposal attached to her name.

The partner returns, opens the same document, sees the associate's proposed edits highlighted (insertions in green, deletions in red strike-through, with the associate's name and timestamp on hover). At the top of the document, a review bar shows "12 pending edits by Vincent". The partner clicks each edit individually and accepts (becomes part of the document) or rejects (vanishes, original text remains). Or uses "Accept all" / "Reject all" with confirmation.

After all edits are resolved, the partner clicks "Export to Word". The document is regenerated as a clean DOCX with all accepted edits applied as final text (no track changes, no comments). She emails it to opposing counsel.

A senior partner re-drafts a section. He proposes edits as well. The associate (assignee) sees both the partner's and her own pending edits, color-coded by author. She can accept her partner's edits without affecting hers.

## Schema delta (UI-facing)

The schema already has:

- `document_versions` with `display_name`, `source` (upload, user_upload, assistant_edit, user_accept, user_reject, generated)
- `document_edits` with `change_id`, `del_w_id`, `ins_w_id`, `deleted_text`, `inserted_text`, `context_before`, `context_after`, `status` (pending, accepted, rejected), `chat_message_id` linkage

What needs adding:

- `document_edits.proposer_user_id` uuid foreign key to auth.users(id) — who proposed this edit
- `document_edits.resolved_by_user_id` uuid foreign key to auth.users(id) nullable — who accepted/rejected
- `document_edits.resolved_at` timestamptz nullable

This enables author attribution and an audit trail.

## Screens

### DOCX viewer (matter document)

Reuse the existing `DocxView` component. Add three modes:

- **Read** (default): shows the document with no edit affordances, accepted edits already merged, rejected edits omitted
- **Propose** (when current user is assignee or shared collaborator and clicks "Suggest edits"): user can highlight + edit text, see only their own pending edits and previously-accepted history
- **Review** (when current user is partner / matter assignee and the document has pending edits): shows all pending edits grouped by author, with accept/reject buttons

Mode is selected via a toggle group in the document toolbar (top of the document, sticky). Default depends on user role and document state.

### Propose mode

When in propose mode:

- Highlighting text shows an inline floating toolbar above the selection: "Replace", "Delete", "Comment" (out of scope for v1), "Cancel"
- "Replace" pops up a small inline input where the user types the new text and presses Enter or clicks Save
- "Delete" immediately marks the selection as a pending deletion (red strike-through, dimmed)
- Inserting new text: cursor placement triggers an "Add text here" tooltip; clicking opens a small input
- Each edit is auto-saved as a pending proposal as soon as the action completes
- The user's pending edits show in green (ins) and red (del) — visually identical to Word track changes

The right rail (in propose mode): a vertical list of "My proposed edits" with each edit summarized ("Page 3: replaced 'governing law shall be...' with 'governing law is...'"). Clicking an edit scrolls and highlights it in the document.

### Review mode

When in review mode:

- A sticky bar at the top of the document shows: "12 pending edits by 2 authors. [Accept all] [Reject all] [Filter by author ▼]"
- Each pending edit is highlighted in the document with author-color (e.g., Vincent's edits in soft purple, Mathias's in soft teal — colors hashed from user id)
- Hover an edit: tooltip shows author name + timestamp + "Accept" / "Reject" buttons
- Click an edit: opens a side popover with full context (before / after), author, time, and large Accept / Reject buttons
- Keyboard: with focus on an edit, `A` accepts, `R` rejects, arrow keys navigate to next edit
- The right rail in review mode: "Pending edits" list grouped by author, each with Accept / Reject inline

When all edits are resolved, the sticky bar shows "All edits resolved" with a "Export to Word" CTA.

### Export to Word

Triggered from the document toolbar or the resolved-edits sticky bar:

- Generates a new DOCX file with all accepted edits applied (rejected omitted) and no track changes markup
- Saves a new `document_versions` row with `source = 'generated'` and a sensible filename like "Acme SPA - Final 2026-05-10.docx"
- Downloads automatically and surfaces a toast "Exported. New version saved."

LibreOffice handles the round-trip: the original DOCX is converted to a structured representation, edits applied, then exported back to DOCX.

### Author colors

Each user's edits are tinted with a color hashed from their user id (deterministic). The palette:

- 12 distinguishable hues, all desaturated to 25% saturation so they don't fight the page text
- Colors meet 4.5:1 contrast against white background
- A small color legend appears in the right rail showing "Vincent (purple), Mathias (teal), ..."

In dark mode, the colors are inverted to higher saturation against the dark page.

### Accepted edit visualization (after acceptance)

Once accepted, the edit is no longer highlighted in the document; the new text is just part of the document. To audit history, a "Show edit history" toggle re-displays accepted edits with a faint underline and author tooltip.

## Component additions

- `<DocxEditModeToggle>` — the toggle group in the toolbar (Read / Propose / Review)
- `<ProposeToolbar>` — the floating toolbar above selected text in Propose mode
- `<ReviewBar>` — the sticky top bar in Review mode with pending count and Accept all / Reject all
- `<EditCard>` — already exists, extend it with author color and resolve actions
- `<EditAuthorLegend>` — color-key in the right rail
- `<ExportToWordButton>` — triggers regeneration and download

Existing `EditCard` and `DocxView` are extended, not replaced. The accept/reject mutations already exist in the API; this brief adds the visual layer.

## Edge cases

**Concurrent edits to the same text**: two lawyers propose conflicting edits to the same passage. Both edits are stored. The partner reviewing sees them as overlapping highlights with hover tooltips showing each. Accepting one auto-rejects the other (with a "this conflicts with X's edit" warning). For the rare three-way conflict, partner manually resolves.

**Pending edit on text that no longer exists** (e.g., another edit deleted the surrounding paragraph): edit becomes "stale". Marked as such with a yellow warning tag in the right rail, "Stale: surrounding text was modified". Partner can still accept (edit applies as a comment) or reject.

**Document with hundreds of pending edits**: virtualize the right-rail edit list. Provide filter by author and by section (collapsible by document heading). Bulk Accept/Reject by author or by section.

**Export with stale edits**: prompt before export "3 edits are stale and may not apply cleanly. Proceed with export? They will be skipped." with confirm.

**Re-uploading a new version of the same document**: existing pending edits become orphaned (referenced to old version). Show banner "This document has been replaced. View previous version? Or re-propose your edits on the new version." Edits do not auto-migrate.

**Authors who left the firm**: their edits remain attributed to their (deleted) user record. UI shows "Former associate" instead of a name, but edits are still resolvable.

**Right of access**: only the matter assignee, partner, and shared_with collaborators can propose or review edits. Other firm users cannot see the document or its edits at all (privilege wall).

## Visual references

Word's Track Changes and Google Docs's Suggest Edits mode are the canonical references. The right-rail edit list mirrors Notion's comment column. The author color hashing pattern mirrors Linear's assignee colors.

## Accessibility

- Edits accept/reject reachable by keyboard (Tab focus on edit, A or R action keys with `aria-label`)
- Author colors paired with text labels (color is never sole indicator)
- Document zoom and text resizing should not break edit overlays (test at 200% zoom)
- Screen readers announce "Pending edit by [author]" when focus enters an edit

## Acceptance criteria

A user with edit access can highlight text and propose a replacement, deletion, or insertion. Edits are saved as pending and visible to other collaborators with author attribution. The matter assignee or partner can review and accept or reject each pending edit. After all edits are resolved, the document can be exported as a clean DOCX. Edit history is auditable. Author colors are consistent across sessions.

## Out of scope

- Real-time co-editing (multiple cursors, simultaneous typing) — heavy version, separate workstream
- Inline comments and threaded discussions on edits — v2 feature
- Edit grouping ("approve as a set") — manual selection workaround for now
- Markdown / rich-text formatting changes (bold, italic) — Word's TrackChanges supports this; v1 covers text-only edits, formatting changes come in v2
- Track changes in PDF documents (most legal review happens in Word; PDF stays read-only)
- Suggesting clause replacements from a clause library / template (separate brief, with firm's playbook)
