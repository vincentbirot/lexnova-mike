# Brief 08 — Admin Panel

## Context

As Lex Nova grows beyond Vincent using Lexnova AI alone to a 5-15-lawyer team, governance becomes essential. Who's in the firm. Who can do what. What integrations are wired. What templates the team uses. What the firm's collective time and matter activity looks like.

This brief introduces an Admin panel accessible only to users with the `admin` role. It centralizes user management, integrations, firm-wide templates, audit log, billing and usage, and overall firm activity. The admin panel does not replace the partner dashboard (Brief 10) which is operational; the admin panel is governance.

## Audience

Firm admin role only. Initially: Vincent. Eventually: managing partners or office manager. Most lawyers should not see this panel.

## Primary user stories

A managing partner adds a new associate to Lexnova AI. She opens admin, clicks "+ Invite user", enters the associate's email and role, sends invite. The associate receives an email with a sign-up link.

An admin offboards a departed lawyer. She clicks the lawyer's row in users, "Disable account". The user can no longer log in. Their existing matters retain their work but show "Former associate" instead of the name.

An admin checks integration status. She sees Gmail connected for 7 of 9 users, Slack workspace connected, R2 active, current Anthropic spend USD 247 this month. She adjusts the firm-wide default tabular model from gemini-flash to gemini-pro.

An admin reviews firm-wide template library. She edits the standard "BOI Application checklist" task template that auto-applies when a new BOI matter is created.

An admin reviews the audit log. She filters to "deletions" in the last 30 days, sees who deleted what.

## Schema delta

New tables / columns:

- `auth.users` schema needs a `role` text column extension via user_profiles or a new `firm_users` table: role enum (`admin`, `partner`, `associate`, `paralegal`, `staff`, `client_external`)
- `firm_settings` (singleton row): firm name, logo URL, default tabular model, default chat model, billing contact, address, billing rate defaults
- `audit_log` table: actor user id, action (create, update, delete), entity type and id, details jsonb, created_at
- `task_templates` and `matter_templates` tables for firm-wide reusable templates
- `integration_configs` per integration (Gmail per user, Slack workspace, etc.) — partial overlap with Brief 06/07 schemas

## Screens

### Top-level layout

Admin panel lives at `/admin`. Sidebar from main app remains. Inside `/admin`, a secondary left-sub-nav with sections:

1. Overview (default landing) — firm dashboard with key metrics
2. Users — user management
3. Integrations — Gmail, Slack, R2, Anthropic, Gemini, others
4. Templates — task templates, matter templates, workflow library
5. Billing & usage — Anthropic / Gemini spend, R2 storage, Supabase tier
6. Audit log — recent significant events
7. Firm settings — firm name, logo, defaults

### Overview tab (`/admin`)

A grid of cards summarizing firm health:

- Active users (count, with breakdown by role)
- Matters this week (created, completed, in-progress)
- Hours logged this week (total billable + non-billable)
- AI spend this month (running total, vs last month)
- Storage used (vs R2 quota)
- Pending invitations (count)
- Integration status (Gmail X/Y connected, Slack: ✓)

Each card is clickable, navigating to the relevant detail tab. Cards refresh on page load with optimistic loading skeletons.

### Users tab (`/admin/users`)

A table of all firm users. Columns: avatar + name, email, role chip, status (active/disabled/invited), last sign-in, matters assigned (count), hours this month, actions menu.

Top of the table: "+ Invite user" primary button, search input, filter by role and status.

Click a row: opens a side panel with full user detail:

- Profile info (read-only)
- Role (editable dropdown)
- Status (Active / Disabled radio)
- Reset password (sends email)
- Force sign-out (invalidates all sessions)
- View activity (link to audit log filtered to this user)
- Disable account (with confirm modal)
- Reassign matters (when disabling, prompt to reassign all matters they own)

Invite user modal: email, role, optional welcome message, send. Creates a Supabase Auth invitation; user receives email; on first login they are guided through profile setup.

### Integrations tab

Card per integration:

- Anthropic (always connected via firm API key from .env; show usage and key obfuscated)
- Gemini (same, optional, can be disconnected if firm uses Anthropic only)
- OpenRouter (optional, advanced)
- Resend (transactional email)
- Gmail (per-user connections; show count connected)
- Slack (workspace connection; show workspace name)
- R2 / Cloudflare (always connected; show bucket and current size)

Each card shows status, last activity, primary action (connect / disconnect / configure).

### Templates tab

Two sections:

**Task templates**: a library of pre-defined task lists for common matter types. Each template has a name, description, applies-to (matter types), and a list of default tasks. Examples:

- "BOI Application — standard checklist" with 14 tasks
- "Share transfer — standard closing checklist" with 22 tasks
- "Lease registration — Land Office checklist" with 8 tasks

When a new matter is created, the user can apply a template, which creates all the tasks.

Admin can edit, duplicate, archive templates. Changes apply only to future applications, not retrospectively to existing matters.

**Workflow library**: same as Mike's existing workflows but at the firm level (not per-user). Admin can promote a personal workflow to firm-wide library.

### Billing & usage tab

Shows current month's spend across providers:

- Anthropic: requests, input/output tokens, USD spend, projected month-end
- Gemini: same
- R2: storage GB, operations Class A and Class B, USD spend
- Supabase: tier, active connections, DB size

Per-user breakdown: top spenders by AI tokens this month (helps identify outlier usage).

Set monthly budget caps with email alerts at 50%, 80%, 100% thresholds.

### Audit log

Reverse-chronological list of significant events:

- User invited / disabled / role changed
- Integration connected / disconnected
- Matter deleted
- Document deleted
- Bulk reassignment
- Firm setting changed

Filter by user (actor), date range, action type, entity type. Export as CSV.

### Firm settings

Form fields:

- Firm name
- Firm address
- Logo upload (replaces Lexnova AI default)
- Time zone (default Asia/Bangkok)
- Default chat model (Claude Sonnet, Claude Opus, Gemini Pro)
- Default tabular model (Gemini Flash, Gemini Pro, Claude Sonnet)
- Default billing rate THB per hour (per-role override)
- Notification defaults (Slack enabled, email enabled, etc.)

Changes save on click of "Save changes" with confirmation toast. Some changes (e.g., default model) take effect immediately; others (e.g., logo) require deploy.

## Component additions

- `<AdminLayout>` — secondary sub-nav layout for admin panel
- `<AdminOverviewCard>` — KPI card for firm metrics
- `<UserTable>` — admin-scoped user list with role and status
- `<UserDetailPanel>` — slide-in panel for user editing
- `<InviteUserModal>` — invitation flow
- `<IntegrationCard>` — per-integration status card
- `<TaskTemplateEditor>` — template editing
- `<UsagePanel>` — spend and quota visualization
- `<AuditLogTable>` — filterable audit log

## Edge cases

**Last admin disabling themselves**: prompt "You are the only admin. Promote another user to admin first." Block the action.

**Disabling a user with running timer**: stop the timer first, with their last entry preserved.

**Disabling a user with pending tasks assigned**: bulk-reassign or unassign before disabling. Prompt admin to choose new assignee.

**Audit log retention**: keep all entries indefinitely (small storage cost). Filter UI handles large volumes.

**Role change effects**: associate→partner: gains access to admin? No, role and admin are orthogonal — admin is a separate flag. Role is for UI / business logic only.

**Multiple admins**: support unlimited. Each admin sees the full panel. Future: add a "super admin" tier if conflicts arise.

## Visual references

Linear's admin panel for the structure and density. Vercel's team settings for the integrations cards. Notion's workspace settings for firm-wide controls.

## Accessibility

- Tables fully keyboard navigable
- Destructive actions (disable user, delete template) require typed confirmation in modal
- Form fields have visible labels and error states

## Acceptance criteria

An admin can invite users, change roles, disable accounts, manage integrations, edit task templates, see firm-wide usage, and review the audit log. Non-admin users do not see /admin in the nav and get a 403 if they navigate to it directly. Critical actions are confirmed and logged. Role changes propagate through the app on next login.

## Out of scope

- Custom roles beyond the predefined set (not in v1; modify the enum if a new role emerges)
- Per-matter access control overrides (matters use shared_with; for now no fine-grained ACL)
- Multi-firm / tenancy (Lexnova AI assumes one firm per deployment in v1)
- White-label theming for client-facing surfaces (no client-facing surfaces yet)
- API rate-limit policy enforcement (we use Anthropic's own caps)
- SAML / SSO for enterprise login (later, when team > 30)
- Subscription billing for the platform itself (this is internal; Lex Nova doesn't charge itself)
