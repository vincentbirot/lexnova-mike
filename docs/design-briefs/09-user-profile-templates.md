# Brief 09 — User Profile and Templates

## Context

Each Lex Nova lawyer has their own way of working. Some prefer Claude Sonnet, others Gemini Pro. Some have personal email signatures and standard greetings. Some have a default checklist they apply to every BOI matter. Some have a personal billing rate that differs from the firm's default. Today, none of this is captured in Lexnova AI: every user looks identical to the system.

This brief introduces a per-user profile with: personal information, model preferences, billing rate, signature, personal task templates, personal workflow templates, notification settings, and connected integrations (Gmail). It complements the firm-wide admin panel: admin sets defaults, each user overrides.

## Audience

Every Lex Nova lawyer.

## Primary user stories

A new associate first signs in. She is taken through a 3-step onboarding: profile (name, role, photo), preferences (default chat model, time zone), connect Gmail. Each step is skippable and revisitable later.

A partner sets her email signature. Every email she sends from Lexnova AI auto-appends her signature.

A lawyer creates a personal task template "Standard SPA closing checklist" with 18 tasks. Next time she creates a Share Purchase matter, she applies her template and all 18 tasks pre-populate.

A senior associate views his profile and sees this month's stats: 47.5 billable hours, 12 matters worked on, 89 chat sessions. Nice ego boost (and also useful self-reflection).

A user changes their default chat model from Claude Sonnet to Gemini Pro because they're testing Gemini's bilingual quality on Thai documents.

## Schema delta

Existing `user_profiles` already has display_name, organisation, tabular_model, claude_api_key, gemini_api_key. Extend with:

- `default_chat_model` text — e.g., "claude-sonnet-4-6", "gemini-2-pro"
- `email_signature` text (HTML)
- `time_zone` text — IANA timezone, default "Asia/Bangkok"
- `phone` text nullable
- `role_label` text — display name for role like "Senior Associate, Corporate"
- `billing_rate_thb_per_hour` integer nullable (overrides firm default from Brief 08)
- `notification_email_enabled` boolean default true
- `notification_slack_enabled` boolean default true
- `avatar_url` text nullable

New tables:

- `user_task_templates` — per-user reusable task lists (matches structure of firm task templates from Brief 08)
- `user_workflow_overrides` — per-user customizations to firm-wide workflow templates

## Screens

### Profile route (`/account` or `/profile` — keep existing path if any)

The existing `/account` page becomes the profile page. Single-page layout with sub-tabs:

1. **Profile** (default) — personal info
2. **Preferences** — model selection, theme, time zone, notifications
3. **Templates** — personal task and workflow templates
4. **Connections** — Gmail, Slack identity, optional API keys
5. **Billing rate** — per-user rate override
6. **Stats** (read-only) — usage stats this month and all-time

### Profile tab

Form fields:

- Avatar (upload, displayed at 96px, falls back to initial-color hash)
- Display name
- Role label (e.g., "Partner, Corporate Practice")
- Email (read-only, change via Supabase Auth)
- Phone (optional)
- Time zone (dropdown, default Asia/Bangkok)
- Email signature (rich-text editor)

"Save changes" sticky bottom-right of the form.

### Preferences tab

- Default chat model: dropdown (Claude Sonnet / Claude Opus / Gemini Pro / Gemini Flash / OpenRouter custom)
- Default tabular model: same
- Theme: System / Light / Dark
- Time zone: covered in profile, but also here as quick toggle
- Notifications: per-event checkboxes (email and slack independently)
  - Task assigned to me
  - Matter assigned to me
  - Email arrived for matter I'm on
  - Daily digest at 8am
- Keyboard shortcuts: read-only reference list, future option to customize

### Templates tab

Two sub-sections:

**Task templates**: list of personal task templates with name, count of tasks, last used date. Click to edit.

Editing a template: a side panel with the template name, description, list of task definitions (title, default due-date offset, default assignee), Add / Delete / Reorder. "Save" saves changes; "Use template" applies it to a matter (matter picker dropdown).

"+ New template" creates a blank one. "Duplicate from firm template" lets the user start from a firm-wide template (Brief 08) and customize.

**Workflow templates**: similar to existing Mike workflows with prompt + columns config; user can list, create, edit, archive.

### Connections tab

Each integration as a card:

- **Gmail**: covered by Brief 06. Connect / Reconnect / Disconnect.
- **Slack identity**: shows linked Slack user (auto-matched by email). If unmatched, shows "Slack identity not linked" with manual link option (search and select Slack username).
- **Custom API keys** (advanced): optional fields for personal Anthropic / Gemini / OpenRouter keys that override the firm's default. Most users leave these empty.

### Billing rate tab

- Default rate from firm settings, displayed as read-only "Firm default: THB X / hour"
- Override field: "Your rate (per hour, THB)" — empty means use firm default
- Effective from date picker — useful for tracking rate changes over time
- Historical rates table: a small table of past rates with effective dates (read-only audit trail)

### Stats tab

Read-only personal dashboard:

- This month: hours logged, matters worked, chats started, documents uploaded
- All time: same columns, plus "Top 5 matters by hours" list
- Streaks: longest consecutive days using Lexnova AI (cute gamification, optional)
- AI usage: tokens consumed, cost (visible only to user themself, plus admin)

### Onboarding flow (new user, first sign-in)

A 3-step wizard, can be skipped:

1. Welcome screen: "Welcome to Lexnova AI, [first name]" with brief intro and "Get started" button
2. Profile: name, role label, photo upload (skippable)
3. Connections: optional Gmail connect, optional Slack identity link (skippable)
4. Done: "You're set up. Open the assistant to get started." with primary CTA "Open Assistant"

## Component additions

- `<ProfileTabs>` — sub-tab navigation
- `<EmailSignatureEditor>` — rich-text editor (Tiptap, same as workflow editor)
- `<TaskTemplateEditor>` — template editing (shared with admin Brief 08)
- `<NotificationPreferences>` — per-event checkboxes
- `<UserStatsDashboard>` — read-only stats panel
- `<OnboardingWizard>` — first-sign-in flow

## Edge cases

**Email signature too large**: cap at 5000 characters or 50KB. Show character count.

**Avatar upload size**: limit to 2MB, auto-resize on server to 512x512.

**Time zone change after entries logged**: existing time entries keep their stored timestamps; display reflects new time zone for new views.

**Default model change mid-conversation**: existing chat continues with previous model; new chats use the new default.

**API key override conflict**: if user provides personal key AND firm key is also set, personal key wins. Document this clearly in the UI.

**Disabled account access to profile**: a disabled user can still access their profile read-only to download their own data (export their time entries, chat history) per data-portability obligations.

## Visual references

Linear's user settings sub-tab pattern. Notion's profile + preferences split. Slack's profile editor for the rich-text signature.

## Accessibility

- Form fields have visible labels and error states
- Avatar upload supports keyboard (focus the upload button, Enter to open file picker)
- Theme toggle reflects user choice immediately, system option respects OS preference

## Acceptance criteria

A user can edit their profile, preferences, templates, connections, billing rate, and view stats. Changes persist and reflect across the app. New users see the onboarding wizard once. The email signature appears at the bottom of emails sent via Brief 06. Personal task templates can be applied to new matters. Notification preferences gate email/Slack sends.

## Out of scope

- Multi-language UI (Thai locale comes later — design ready for it)
- Profile photo cropping UI
- Two-factor authentication setup (Supabase Auth handles, no UI in v1)
- API key rotation reminders
- Personal calendar integration (later)
- Team-shared personal templates (out of scope; firm-wide templates handle sharing)
