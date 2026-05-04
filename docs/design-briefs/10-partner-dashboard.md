# Brief 10 — Partner Dashboard

## Context

A partner at Lex Nova manages 30-50 active matters across 5-10 associates. Today, that oversight is fragmented: matter status in their head, time logged in spreadsheets, client communication in Gmail, decisions in Slack. They cannot answer "what is at risk this week" without burning an hour.

The partner dashboard is the single screen that answers the partner's daily question: where is everything, what needs my attention, what's at risk. It pulls from every other feature: matter status (Brief 02), tasks (Brief 03), time clock (Brief 04), Gmail (Brief 06), Slack (Brief 07). It does not introduce new data; it surfaces patterns from existing data.

This is the highest-leverage feature for partners. It is the screen they open every morning. It is what justifies Lexnova AI to the rest of the firm.

## Audience

Partners and senior practitioners with oversight responsibilities. Junior associates can also view but with their own scope only. Admin role sees an "All firm" toggle to see firm-wide.

## Primary user stories

A partner opens Lexnova AI in the morning. The dashboard greets her with a personalized panel:

- "Good morning, Vincent. You have 3 matters at risk and 2 tasks overdue."
- A list of "At risk matters" — those awaiting client more than 7 days, with overdue tasks, or approaching budget cap
- A list of "Today's tasks" — her tasks due today
- A list of "Recent activity" — last 24h: documents uploaded, tasks completed, emails arrived

She decides what to do today in 60 seconds.

A managing partner opens the firm-wide view (admin role). She sees the same patterns but aggregated across all lawyers: total firm hours this week, top matters by activity, matters with no recent activity (stale), associates at capacity vs. underloaded.

A partner sees that "Acme — Q2 Restructuring" has burned 78% of its fee cap with 3 weeks left. She gets ahead of the budget conversation with the client.

A partner notices an associate has not logged any time in 4 days. She nudges them gently or asks if they're stuck.

## Architecture

The dashboard is read-only — it queries existing data and renders panels. Heavy joins; cache aggressively (most metrics refresh every 5 minutes). Real-time updates for high-priority items (new task assigned, email arrived) via WebSockets if the architecture supports, otherwise polling every 60s when tab is active.

Personal scope (default): matters where the partner is assignee or in shared_with. Firm scope (admin only, toggleable): all matters.

## Screens

### Dashboard route (`/dashboard`)

A new top-level route, accessible from sidebar (above Assistant — make it the default landing page when user is partner or admin role).

Layout: hero greeting bar at top, then a grid of panels below.

### Hero greeting

Top of the page. Personalized:

> Good morning, Vincent. You have 3 matters at risk, 2 tasks overdue, and 5 emails awaiting your reply.

Each clause is a clickable link that scrolls to / filters the relevant panel. Right side of hero: a quick "Start a chat" button (opens new general chat) and "Create matter" button.

Below hero: a single row of 4 KPI cards:

- Active matters (count, with sparkline trending up/down vs last week)
- Hours logged this week (vs target if set)
- Tasks due today (with overdue count in red)
- AI runs this week (and remaining quota if budget cap exists)

Each card clickable to drill in.

### Panel: Matters at risk

A card showing the most concerning matters. Risk signals:

- Awaiting client > 7 days (no inbound email or Slack activity)
- Overdue tasks (any task past due date, status not done)
- Approaching budget cap (>75% of estimated fee)
- High activity but no time logged (matters with documents and chats but no time entries this week)

Each row: matter name (clickable), risk signal as a chip, last activity timestamp, primary CTA per signal type ("Send nudge", "Review tasks", "Check budget").

Default shows top 5; "View all" expands to filter and sort.

### Panel: Today's tasks

Same data as Brief 03's My Tasks sidebar, but expanded. Tasks due today first, then due this week. Each task row clickable to open the task detail. Inline complete checkbox.

### Panel: Recent activity

Last 24 hours of significant events across the partner's matters:

- Documents uploaded (with thumbnail and uploader)
- Tasks completed
- Emails arrived (linked threads from Brief 06)
- Slack messages with mentions (when implemented)
- Chats started by team members on shared matters

Reverse-chronological feed. Click any item to navigate to it.

### Panel: Team capacity (firm-wide view, admin only)

A horizontal bar chart of associates by hours logged this week:

- Each row: associate name, avatar, weekly hours bar (filled to capacity, color-coded: green <40h, yellow 40-50h, red >50h)
- Sort by hours descending
- Click a bar to navigate to that associate's timesheet

Below the chart: a list of "Underloaded" (associates with <20h this week — possibly available for new matters) and "Overloaded" (associates with >50h — possibly at burnout risk).

### Panel: Awaiting client

List of matters where the last outbound email was sent by Lex Nova more than 3 days ago and no reply received.

Each row: matter name, days since last reply, last subject line, "Send nudge" action that pre-drafts a polite follow-up email.

### Panel: AI usage and spend

A small card summarizing:

- This week: chats started, tabular reviews run, total tokens consumed, USD spend
- Vs last week: percentage delta
- Per-matter breakdown (top 5 by spend) — useful to identify runaway costs

### View modes

Toggle at top right of dashboard:

- "My matters" (default for partners) — only matters where I am assignee or in shared_with
- "All firm" (admin only) — every matter, every lawyer

Switching mode reloads all panels with the new scope.

### Date range

Default "This week". Changeable via a date range picker affecting the rolling-window panels (recent activity, hours, awaiting client, etc.).

## Component additions

- `<PartnerDashboardLayout>` — the page layout with hero, KPI row, panel grid
- `<HeroGreeting>` — personalized greeting with linked clauses
- `<KpiCard>` — title, big number, sparkline, trend indicator
- `<RiskPanel>` — matters-at-risk list
- `<RecentActivityFeed>` — chronological event list
- `<TeamCapacityChart>` — horizontal bar chart with thresholds
- `<AwaitingClientList>` — filtered matter list with nudge action
- `<AiSpendCard>` — usage and cost summary

## Empty states

- New partner with no matters: "Welcome. Once you have matters and team activity, your dashboard will fill in here." with CTAs to create a matter and explore Mike features
- All caught up (no risk signals, no overdue): "Inbox zero, dashboard zero. Solid morning." with subtle celebration emoji and the date

## Edge cases

**Slow first load**: dashboard requires many aggregate queries. Show panel-level skeletons. Don't block the page on any single panel — render whichever finish first.

**Stale data due to cache**: show last-refresh time in a small footer. Manual refresh button that bypasses cache.

**Partner with no team capacity (firm-wide view in solo or 2-person firm)**: the panel hides or shows a friendly note instead of an empty bar chart.

**Risk signals overlap**: a matter could be on multiple lists (overdue tasks AND awaiting client AND over budget). Show in the most severe list only, with a +N badge for additional signals.

**Personalization data lag**: when matter is created or task assigned, dashboard data may not reflect for up to 5 minutes due to caching. Force-refresh button addresses this.

**View mode permissions**: associates cannot toggle to firm-wide view. The toggle UI hides for non-admin / non-partner roles.

## Visual references

Linear's "Inbox" and "My Issues" combined. Pipedrive's deal pipeline + activity feed. Figma's organization dashboard for the team capacity visualization. Asana's My Tasks.

## Accessibility

- Charts must have textual data table alternatives (`aria-describedby` linking to a hidden table)
- Sparklines have title attributes describing the trend
- Personalized greetings update for screen readers when refreshed

## Acceptance criteria

A partner sees a personalized dashboard on `/dashboard`. The dashboard surfaces matters at risk, today's tasks, recent activity, and team capacity (admin view). Risk signals are accurate and actionable. The dashboard is the default landing for partner / admin roles. Performance is acceptable: full first paint under 2s on a typical Lex Nova dataset (50 matters, 1000 tasks, 5000 time entries).

## Out of scope

- Customizable widget grid (partners can't rearrange panels in v1; fixed layout)
- Predictive analytics (estimating completion times, predicting matter outcomes) — this is real product work for v3+
- Cross-firm benchmarks (Lex Nova compared to industry) — out of scope
- Drill-down explainers on AI cost (per-prompt cost analysis) — Brief 08 admin handles aggregate usage
- Mobile-optimized dashboard (desktop only for v1; mobile responsive layout works but is secondary)
