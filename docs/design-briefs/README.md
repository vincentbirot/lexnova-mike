# Lexnova AI — Design Briefs

This folder contains design specifications for every feature in the Lexnova AI build plan. Each brief is intended to be consumed by a design AI tool (Claude with artifact tool, v0, Lovable, or similar) to produce wireframes, then by Claude Code to implement the screens.

## How to use these briefs

Read in order. Each brief assumes the reader has read `../design-system.md` first.

For the design AI: produce wireframes following the screen-by-screen descriptions, using Mike's existing visual language (Inter sans + EB Garamond serif, neutral palette with azure blue accent, shadcn/ui + Lucide icons). When in doubt about styling, mirror the corresponding existing screen referenced in each brief.

For Claude Code: read the design AI's output (wireframes saved as JSX or images) plus the brief, then produce the implementation as a focused PR.

## Build order

The order below is the proposed sprint order. Each brief is self-contained and can in principle be built independently, but the order respects dependencies (e.g., admin panel depends on per-user profiles, partner dashboard depends on time clock and tasks).

| # | Brief | Sprint | Estimated build time | Depends on |
|---|---|---|---|---|
| 00 | Platform rebrand: Mike → Lexnova AI | Pre-sprint (foundation) | 4-6 hours | none |
| 01 | Concurrent request queue | Sprint 1 | 1-2 days | rebrand |
| 02 | Matter status + assignee | Sprint 1 | 1 day | rebrand |
| 03 | Tasks per matter | Sprint 1 | 1-2 days | 02 |
| 04 | Time clock + timesheet | Sprint 2 | 2 days | 02 |
| 05 | DOCX collaboration (light) | Sprint 2 | 2 days | rebrand |
| 06 | Gmail integration | Sprint 3 | 3-4 days build + ~7 days OAuth verification | 02 |
| 07 | Slack integration | Sprint 4 | 2-3 days | 02 |
| 08 | Admin panel | Sprint 5 | 2-3 days | 02, 04, 09 |
| 09 | User profile + templates | Sprint 5 | 1-2 days | rebrand |
| 10 | Partner dashboard | Sprint 6 | 2 days | 02, 03, 04, 06, 07 |
| 11 | Firm-wide RAG memory | Sprint 7 | 1-2 weeks | 02 |

## Cross-cutting principles

These apply to every brief:

**Lawyer-first language.** The audience is a Bangkok-based corporate lawyer at Lex Nova Partners. Use legal vocabulary throughout: matter, client, engagement, retainer, billable hour, conflict, privilege. Never use product-marketing language like "supercharge," "leverage," or "unlock."

**Bilingual readiness.** Every UI string should be wrappable in an i18n function call later. Avoid baking English strings into deep code; use a constant or a translation key. The first translation will be Thai. Reserve 30% extra horizontal space for Thai (it is not significantly longer than English but uses more vertical space and different character widths).

**Keyboard-first.** Lawyers navigate fast. Every primary action needs a keyboard shortcut. Document shortcuts in tooltip text and a `?`-triggered shortcuts modal.

**Empty states sell the feature.** When a section has no data yet, the empty state must explain what the section is for and offer a clear action to populate it. Never leave a screen blank with just "No data."

**Privilege awareness.** Lex Nova handles confidential matters. Any feature that could leak across matters (search, dashboards, AI memory) must respect matter access boundaries. Surface this visually with locks, restricted-icons, or "Confidential — limited to assigned team" banners where relevant.

**Performance perception.** Even when the underlying API is slow, the UI should feel fast. Use optimistic updates for actions like creating a matter, assigning a task, starting a timer. Show the new item immediately, reconcile when the server responds, revert with a toast on failure.

## Glossary

- **Matter** — the firm's word for a unit of legal work for a client. Replaces "project" everywhere in the UI. Internally in the database the table is still `projects` for now (renamed only at the API/UI boundary). Migration to `matters` table name is a separate workstream not covered here.
- **Assignee** — the lawyer who owns the matter. Each matter has exactly one primary assignee, plus optional collaborators (the existing `shared_with` jsonb array).
- **Task** — a checklist item attached to a matter, with optional due date and assignee.
- **Timer** — an active time-tracking session, billable or non-billable, attached to a matter.
- **Run** — a single AI request from chat or tabular review. Runs queue and execute asynchronously.
- **Workspace** — synonym for the lawyer's view of all their assigned matters and tasks. Not to be confused with "matter" itself.
- **Firm** — the Lex Nova organization. In the long run, a single Lexnova AI deployment serves one firm.

## What's deliberately NOT in this design pass

- Mobile-first redesign (we keep desktop-first, mobile responsive but secondary)
- Calendar integration UI (Calendar comes after Slack)
- Document drafting / generation UI (separate workstream)
- Billing / invoicing screens (use existing PMS like Clio or Amberlo for that)
- Conflict checking workflow (separate workstream, requires conflict database)
- Court filing integrations (Thai e-filing comes much later)
