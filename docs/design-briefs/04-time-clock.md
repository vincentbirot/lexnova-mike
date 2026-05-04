# Brief 04 — Time Clock and Timesheet

## Context

Lex Nova bills clients by time. Every billable hour must be captured. Today the firm relies on lawyers' memory and end-of-week reconstructions, which lose 15-25% of billable time per industry data. The single most valuable feature Lexnova AI can add for revenue is an integrated timer that captures time as the work happens.

This brief introduces a per-matter (and optionally per-task) timer, manual time entries for retrospective capture, a personal weekly timesheet, and a monthly billable summary per matter. It feeds the partner dashboard (Brief 10) with hours-vs-budget signals.

## Audience

Every Lex Nova lawyer who bills time. Partners use the rollups for invoicing and budget oversight; associates use the timer daily.

## Primary user stories

An associate starts work on the Acme matter. He opens the matter, clicks the timer button, types a brief description ("reviewing draft SPA, redlines from Sidley"), and starts the timer. He works for 47 minutes, then stops. The 0:47 entry saves automatically, marked billable.

A partner returns from a client lunch. She did not start a timer at the restaurant. She opens the matter, clicks "+ Add manual entry", picks the start time (12:30) and end time (14:15), types "client meeting at Issaya, BOI application discussion", saves. The 1:45 entry appears in her timesheet.

An associate forgets to stop a timer overnight. The system auto-stops timers that run past 8 hours and asks for confirmation: "Your Acme timer ran for 8 hours. Was this accurate?" with options to set a custom duration, accept 8 hours, or delete.

A partner runs the monthly close. She opens "Time" in the sidebar, sees her week-by-week totals for the month, exports a CSV per matter for invoicing.

A lawyer wants to attach time to a specific task (Brief 03). When starting a timer on a matter, an optional task picker lets her pin the timer to a task. The time entry rolls up under both the matter (for billing) and the task (for closure tracking).

## Schema delta

New table `time_entries`:

- `id` uuid primary key
- `matter_id` uuid foreign key to projects(id) on delete cascade
- `user_id` uuid foreign key to auth.users(id)
- `task_id` uuid foreign key to tasks(id) nullable
- `started_at` timestamptz not null
- `ended_at` timestamptz nullable (null = currently running)
- `duration_minutes` integer (computed via trigger when ended_at set; null if running)
- `description` text
- `is_billable` boolean default true
- `hourly_rate_thb` integer nullable (snapshot of rate at time of entry, optional)
- `billed_at` timestamptz nullable (set when entry is included in an invoice)
- `created_at`, `updated_at` timestamptz

Indexes: (user_id, started_at desc), (matter_id, started_at desc), (matter_id, billed_at) for invoicing queries. Unique partial index ensuring at most one running timer per user (where ended_at is null).

## Screens

### Persistent timer widget (top of every page)

A small floating timer widget appears in the top-right of the page header on every screen, only when a timer is running. It shows:

- The matter name (truncated)
- The elapsed time in HH:MM format, updating every second
- A pulsing red dot to indicate running
- A "stop" button (`Square` icon)
- Click expands to show the description, task association, and a quick-edit popover

When no timer is running, the widget is hidden but a small "+ Start timer" button replaces it for the current matter (only on matter-related pages).

### Matter detail — Time tab (new tab on matter detail page)

Add a "Time" tab to the matter detail page (after Tasks, before Assistant). Shows:

- A header bar with: "+ Start timer" (if no timer running) OR active timer card (if running for this matter), and "+ Add manual entry" secondary button
- Filters: date range picker, billable/non-billable, person (visible only to admin / partner)
- Time entries list grouped by date (today, yesterday, earlier this week, last week, older)
- Each entry: start time, duration, description, billable chip, person avatar, edit / delete actions
- A footer summary: "Total: 12h 45m | Billable: 11h 30m | This month: 47h 12m"

### Personal timesheet (`/time`)

A new top-level route /time accessible from the sidebar (after My Tasks, before Workflows in the nav order). Shows the current user's time entries across all matters:

- Header with date range selector (defaults to "This week")
- Quick-jump tabs: Today / This week / Last week / This month / Custom
- Entries grouped by day, each day showing the day's total at the top
- Each entry shows the matter name (clickable), description, duration, billable chip
- Entries are inline-editable: click description to rename, click duration to adjust
- A right rail summary: total hours, billable hours, target (if set in profile), variance

### Manual entry modal

When clicking "+ Add manual entry":

- Modal opens with: date picker (default today), start time (HH:MM), end time (HH:MM), description (multi-line), task picker (optional, filtered to current matter), billable toggle
- Validates that end > start, end <= now (no future entries)
- Save creates the entry; toast confirms

### Stopping the timer

Clicking the stop button on the running timer widget:

- Stops immediately
- Slides down a small confirmation sheet from the top: "1h 23m logged. [Description input, prefilled if user typed earlier]. Billable / Non-billable toggle. Save / Discard."
- Save: closes the entry. Discard: deletes the running timer (with a confirm if duration > 5 minutes).

### Monthly close (per matter)

On the matter detail Time tab, a "Generate timesheet" button at the right of the filter row produces a downloadable PDF / CSV of all entries in the selected date range, formatted for invoicing. Includes: matter name, client name (TBD field), each entry with date / lawyer / duration / description / rate / amount, totals at the bottom.

## Component additions

- `<RunningTimerWidget>` — the floating timer in the page header
- `<StartTimerButton matterId={...}>` — variant of the widget when no timer is running
- `<TimeEntryRow entry={...}>` — single row in time entry lists
- `<ManualTimeEntryModal>` — the manual entry modal
- `<TimesheetSummary>` — the rollup summary box (totals, billable, target)
- `<DateRangePicker>` — reused across timesheet, matter time tab, and dashboards
- `<TimeEntryEditPopover>` — inline-edit popover for description, duration, billable

## Edge cases

**Multiple timers**: forbidden. If a user starts a second timer, the first auto-stops with a "previous timer stopped" toast.

**Timer running across days**: if a timer crosses midnight, the entry's started_at is the actual start (yesterday), ended_at is the actual end (today). Display in the timesheet under the day it ended. (Or under both with a "spans midnight" indicator. v1 picks one, recommend "ended day".)

**Description required**: warn if empty on stop, but allow saving without description. Empty descriptions show as italic "(no description)" in lists.

**Timer started, then app closed**: timer continues server-side. On next app load, the running timer widget re-appears with the correct elapsed time.

**Timer started on Mac, lawyer switches to phone**: timer state is server-authoritative; the phone shows the same running timer.

**Editing a billed entry**: once an entry has billed_at set (locked into an invoice), editing should be blocked. Lawyer must "unbill" first (a separate flow, partner-only, requires reason).

**Negative or zero duration entries**: not allowed. Validation rejects.

**Lunch breaks**: lawyers stop the timer, take lunch, restart with a new entry. The system does not auto-pause.

## Visual references

The running timer widget visually resembles Toggl's floating widget. The timesheet day-grouped layout resembles Harvest. The matter Time tab table resembles Clio's time entries. The blend of timer + manual entry + matter rollup is the core of any time-tracking app — there is no need to invent new patterns.

## Accessibility

- Timer elapsed time updates announced once per minute, not per second, for screen readers
- Inline editing is keyboard accessible (focus + Enter to edit, Esc to cancel, Enter to save)
- The "Stop" button has `aria-label="Stop timer for [matter name]"` with the elapsed time

## Acceptance criteria

A timer can be started, displayed persistently across page navigation, stopped, and saved as an entry. Manual entries can be added retrospectively. Entries appear on the matter Time tab and on the personal timesheet. Per-matter monthly totals are computable. Entries can be edited and deleted (unless billed). The running-timer widget is visible on every page when active.

## Out of scope

- Invoice generation (export to CSV/PDF only; firm uses external billing software like Clio or Amberlo for invoicing)
- Per-task budget tracking with alerts (later, when fee caps are added)
- Approval workflow for time entries (partner approves before billing) — not in v1
- Timer reminders / nudges ("you stopped working 30 min ago, restart timer?") — Brief 10 dashboard handles overall hours visibility
- Multi-user billing rates per matter (each user has one default rate; per-matter rate override is later)
- Time entry comments or discussion
- Calendar integration (auto-detect time spent in meetings) — out of scope, requires Calendar integration
