# Brief 03 — Tasks Per Matter

## Context

Lawyers manage their work day in tasks: "Send first draft to Acme by Friday", "Review counsel's redlines", "Confirm escrow agent details", "File NS3K upgrade petition". Today Lexnova AI has no concept of tasks. Lawyers either remember tasks (poorly), keep separate to-do lists (fragmented), or chase emails (lossy).

This brief introduces a task list per matter, a "My Tasks" cross-matter view in the sidebar, and a daily-overdue surfacing. Tasks are the core unit of operational accountability. They are how a partner sees who is doing what and when. They are the input to the time clock (a timer can be started against a task) and to the partner dashboard (overdue tasks are the most important signal).

## Audience

Every Lex Nova lawyer. Partners use tasks to delegate; associates use tasks to track what is owed.

## Primary user stories

A partner reviews a matter and identifies three things to do. He clicks "+ Add task" three times in a row, types each task, sets a due date and assignee for each. The associate sees the new tasks immediately on their next page load.

An associate finishes a task and clicks the checkbox. The task is marked done, struck through, moves to a "Completed" section at the bottom of the list, and a faint celebratory toast says "Task complete."

A lawyer opens her "My Tasks" view in the sidebar. She sees a list of every task assigned to her across all matters, grouped by due date (Overdue, Today, This week, Later). Overdue items show in red.

A senior partner asks "what does Vincent owe Acme this week?" and filters Tasks by assignee = Vincent and matter = Acme. He sees the three open items.

A task is dependent on another (e.g., "send invoice" can only happen after "client signs final agreement"). Today this is captured by ordering tasks but not by enforced dependencies (out of scope for v1).

## Schema delta

New table `tasks`:

- `id` uuid primary key
- `matter_id` uuid foreign key to projects(id) on delete cascade
- `created_by_user_id` uuid foreign key to auth.users(id)
- `assignee_user_id` uuid foreign key to auth.users(id) nullable
- `title` text not null
- `description` text nullable (longer notes)
- `due_date` date nullable
- `status` text enum: `open`, `in_progress`, `done`, `cancelled`. Default `open`.
- `completed_at` timestamptz nullable (set when status moves to done)
- `position` integer (manual sort order within a matter)
- `created_at`, `updated_at` timestamptz

Index on (matter_id, status, position). Index on (assignee_user_id, status, due_date) for the My Tasks view.

## Screens

### Tasks tab on the matter detail page

The matter detail page currently has tabs: Documents, Assistant, Tabular Reviews. Add a new tab "Tasks" between Documents and Assistant (so the order becomes: Documents, Tasks, Assistant, Tabular Reviews).

The Tasks tab shows:

- A header bar with a "+ Add task" primary button on the right
- An optional filter row: assignee dropdown, status dropdown (default "Open + In progress")
- Task list grouped into sections: Overdue (red header), This week, Later, Completed (collapsed by default)
- Each task is a row with: checkbox (to mark done), title (clickable for detail), due date chip, assignee avatar, status chip if not "open", a `MoreHorizontal` row-actions menu

Inline task creation: "+ Add task" reveals a single-row inline form below the header: a text input for the title, a small calendar picker for due date, an assignee picker, and a "Create" button. Enter on the title input also creates. Esc cancels.

Task detail (when clicking a task title): a side panel slides in from the right (modal-like but anchored to the right edge, leaves the matter visible behind). The side panel shows:

- Title (editable inline, click to edit)
- Description (editable inline, supports basic markdown, multi-line)
- Due date (calendar picker)
- Assignee (assignee popover from Brief 02)
- Status (status dropdown: open, in_progress, done, cancelled)
- Created by + Created date (read-only)
- A small comment thread at the bottom (out of scope for v1, leave the section ready in the data model)
- Actions footer: "Delete task" (destructive, with confirm) and "Close" button

### My Tasks (sidebar)

In the sidebar, below the primary nav, add a new section "My tasks" with a count badge showing total open tasks assigned to the current user. The section is collapsible. Expanded, it shows up to 8 tasks grouped by due date:

- Overdue (red dot)
- Today (amber dot)
- This week (gray dot)
- Later (no dot)

Each task in the sidebar shows: title (truncated to 30 chars), matter name (very small, 11px, `text-muted-foreground`), and the due date short-form (e.g., "Tue", "Jul 14"). Click navigates to the task in its matter.

Below the 8 visible tasks, a "View all" link opens a full-page My Tasks view at `/tasks` (a new route) with all tasks, full filters, grouped the same way.

### My Tasks full page (`/tasks`)

A standalone page that lists all tasks across all matters where the current user is the assignee. Same filter bar as the matter-level Tasks tab plus a matter filter. Same grouping (overdue, today, this week, later, completed). No "+ Add task" button here (tasks are always created in the context of a matter).

### Status indicators

- Open: no chip, default appearance
- In progress: small "In progress" chip in soft blue
- Done: checkbox checked, title with strike-through, row dims to 60% opacity, moved to "Completed" section
- Cancelled: small "Cancelled" chip in muted gray, struck through, dimmed

### Empty states

- Matter Tasks tab with no tasks: centered illustration of a checklist icon, "No tasks for this matter yet." subtitle "Tasks help track what is owed and by when." primary CTA "+ Add first task"
- Sidebar My Tasks with no tasks: small italic text "No tasks assigned to you" with a smiley `:)` (no CTA, this is a happy state)
- /tasks page with no tasks: same as sidebar, scaled up

## Component additions

- `<TaskList matterId={...}>` — the matter-level task list with grouping and inline create
- `<TaskRow task={...}>` — single task display, checkbox + title + due + assignee + actions
- `<TaskDetailPanel task={...}>` — slide-in right panel for editing a task
- `<DueDatePicker value={...} onChange={...}>` — calendar picker with quick options ("Today", "Tomorrow", "Next week")
- `<MyTasksSidebar>` — sidebar section showing the user's tasks
- `<MyTasksPage>` — full /tasks route page

## Edge cases

**Overdue tasks**: any task with due_date < today and status not in (done, cancelled). Surface in red. Send a daily summary email at 8am Bangkok time (Brief 06).

**Reassigning a task**: previous assignee notified next time they load the app (in-app toast). Once Brief 06 lands, also via email if the partner setting allows.

**Bulk operations**: not in v1. Tasks are usually managed individually.

**Completing a task with an active timer (Brief 04)**: prompt "Stop the running timer too?" — yes/no.

**Deleting a task with billable time logged against it (Brief 04)**: do not allow deletion. Disable the delete button with tooltip "Cannot delete a task with logged time. Cancel it instead."

**Task due dates and timezones**: store as date (no time component). Display in the user's local timezone (Lex Nova is Bangkok, so UTC+7). "Today" means today in Bangkok.

**Completed tasks accumulating**: after 90 days in completed status, move to an archive (still queryable but not loaded by default in the matter Tasks tab). Improve performance for matters that have hundreds of completed tasks.

## Visual references

The matter Tasks tab structure mirrors Linear's issue list within a project. The task detail side panel mirrors Linear's issue detail panel (slide-in right, edit-in-place fields). The "My Tasks" sidebar section mirrors Linear's "My Issues" or Asana's sidebar inbox. The strike-through-on-complete is universal.

## Accessibility

- Checkboxes have `aria-label="Mark [task title] as done"`
- Date pickers fully keyboard accessible
- Group headers ("Overdue", "Today") use semantic h3 elements
- Color (red for overdue) accompanied by an icon and "Overdue" label

## Acceptance criteria

A task can be created, viewed, edited, completed, and deleted. Tasks appear on the matter detail page in a Tasks tab. Tasks appear in the sidebar grouped by due date. A full My Tasks page shows all assigned tasks across matters. Filters work. Inline rename and inline due-date change work. Overdue tasks are visually distinct and counted in the sidebar badge.

## Out of scope

- Sub-tasks / hierarchical tasks (use separate tasks linked by reference for v1)
- Task templates ("standard checklist for SPA", "standard checklist for BOI application") — Brief 09 covers per-user templates, including task-list templates
- Task dependencies / blockers
- Recurring tasks
- Task comments and discussion threads
- Task attachments
- Notifications for task changes (covered by Brief 06 + 07 via Gmail/Slack)
