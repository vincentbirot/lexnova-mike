# Brief 07 — Slack Integration

## Context

Lex Nova uses Slack for internal coordination. The firm has channels per matter or per workstream (e.g., `#mathias-barbe-real-estate-acquisition`, `#issarafun-corporate`). Today, those Slack discussions are siloed from Lexnova AI: a partner asking "what did the team decide about the BOI angle on Acme" must search Slack manually. The decisions made in Slack do not feed the assistant's context.

This brief integrates Slack at the matter level. A Slack channel is mapped to a Lexnova matter. Messages in that channel are surfaced on the matter detail page and made available as context to the assistant. Lawyers can also send notifications from Lexnova AI to a matter's Slack channel (matter created, task assigned, document uploaded). Unlike Gmail, Slack runs at the workspace level, so we register a single Slack app for the Lex Nova workspace and all lawyers benefit from one OAuth.

## Audience

Every lawyer using Slack at Lex Nova. Particularly relevant for partners who coordinate across many matters via channels.

## Primary user stories

A partner creates a new matter in Lexnova AI. A toggle "Create Slack channel" is enabled by default. On save, a new channel `#acme-q2-2026` is auto-created in the Lex Nova Slack workspace, the matter team is invited, and the channel is linked.

An associate working on the Acme matter goes to the matter detail page, opens the "Slack" tab, and sees the most recent 50 messages from the linked channel inline. He can also click "Open in Slack" to jump to the channel in his Slack app.

A lawyer asks the assistant "What was decided on the structuring approach for Acme?" The assistant searches the linked Slack channel's messages, finds the relevant discussion, quotes it in the response with author attribution and timestamp.

A partner wants the team to know a new draft is ready. From the matter document page, she clicks "Notify channel" and an automated message posts to `#acme-q2-2026`: "📝 Vincent uploaded 'Acme SPA v3.docx' (acme matter)". The channel members react and discuss in Slack.

## Architecture

A single Slack app registered for the Lex Nova workspace. OAuth scopes:

- `channels:read`, `channels:history` — read public channel messages
- `groups:read`, `groups:history` — read private channel messages (the workspace decides if private channels are linkable)
- `chat:write` — send notifications
- `channels:manage` — auto-create channels for new matters
- `users:read`, `users:read.email` — match Slack users to Lex Nova users by email

The OAuth flow is workspace-level: one admin (Vincent) installs the Slack app for the workspace, and from then on every Lexnova AI user benefits without each having to authorize.

Per-user Slack association: each Lex Nova user's Lexnova AI account is mapped to their Slack user ID by email. This allows attribution ("This message is from Vincent" using Vincent's actual Slack identity).

## Schema delta

New tables:

- `slack_workspace_connection` (singleton row) — workspace OAuth tokens, workspace ID, last sync
- `slack_user_links` — maps Lexnova user_id to Slack user id and email
- `slack_channel_links` — maps matter to Slack channel id, with link_type (auto-created, manually linked)
- `slack_messages_cache` — recent messages per channel, with author, timestamp, text, thread_ts (parent if reply)

## Screens

### Slack workspace connection (admin panel)

Available in the Lexnova AI admin panel (Brief 08), not user-level. A "Slack" card under "Integrations" with:

- "Not connected" state: large "Connect Slack workspace" button → triggers Slack OAuth
- "Connected" state: workspace name and icon, connected since date, last sync time, "Disconnect" button

Only the Lex Nova admin (Vincent or a designated partner) can manage this. Other users see the connected status but no controls.

### Matter Slack tab

Add a "Slack" tab to the matter detail page (after Email, before Assistant). Shows:

- A header bar showing the linked channel: "#mathias-barbe-real-estate-acquisition" with a "Open in Slack" external link icon
- Below: scrolling timeline of recent messages, latest at the bottom (Slack-native order)
- Each message: avatar, name, timestamp, message body (with rendered markdown, mentions, links), reactions count
- Threaded replies indented and collapsible with a "X replies" link
- Top of the timeline: "Load older messages" if more than the cached window
- A small input at the bottom: "Reply in this channel" — typing here and sending posts to Slack as the user (using their Slack user ID)

If the matter has no linked channel, the tab shows an empty state with two CTAs: "Create Slack channel" (auto-creates and links) or "Link existing channel" (channel picker dropdown).

### Matter creation flow with Slack auto-create

In the "+ New matter" modal (Brief 02), a section "Slack channel" with:

- Toggle: "Create Slack channel" (default on)
- Visible-only-when-toggled: channel name (auto-suggested from matter name, e.g., "acme-q2-2026"), public/private radio (default public to match Lex Nova's workspace conventions, but user can override), team members to invite (defaults to the assignee + shared_with users, multi-select)

On submit, the matter is created AND the Slack channel is created and linked AND members invited.

### Notifications from Lexnova AI to Slack

Specific events post to the linked channel:

- Matter created (one-time, at creation): "🎯 New matter: Acme SPA — assigned to Vincent"
- Document uploaded: "📄 Mathias uploaded 'Acme DD Report.pdf'"
- Task created and assigned to someone: "✅ Vincent assigned 'Send draft to opposing counsel' to Mathias, due Jul 12"
- Task marked complete: "✓ Vincent completed 'Review Acme redlines'"
- New chat asked: optional, off by default (could be noisy)
- Tabular review completed: "🧮 Tabular review 'Acme DD' finished, 47 rows extracted"

Each notification is a single message to the matter's linked channel. Each user can mute notifications per matter via a setting; partners can configure firm-wide defaults in admin (Brief 08).

### Slack-to-matter context for assistant

When a lawyer asks the assistant a question on a matter, the assistant has access to the linked channel's recent messages (last 30 days, most relevant retrieved via embedding similarity). The assistant cites Slack messages with author attribution: "Vincent said in #acme-q2-2026 on May 1: '[quoted snippet]' [link to message]".

## Component additions

- `<SlackConnectionCard>` — admin panel connection management
- `<SlackChannelTab>` — matter detail Slack tab
- `<SlackMessageRow>` — single message rendering with avatar, name, body
- `<SlackChannelPicker>` — dropdown for linking existing channels
- `<CreateSlackChannelToggle>` — the matter creation form section
- `<SlackNotificationConfig>` — per-matter or firm-wide settings for which events post

## Edge cases

**Slack workspace disconnected mid-use**: banner at top of any Slack tab "Slack disconnected. Reconnect in admin to restore." Notifications queue and retry.

**Matter has no linked channel**: features that require it (Slack tab content, send notification) gracefully disabled with an inline prompt to link.

**User in Lexnova AI is not in the linked Slack channel**: messages still readable on the Slack tab (Lexnova AI uses workspace-level read access). Replies sent as that user fail with toast "You are not a member of this channel. Join in Slack first."

**User has no Slack account**: messages display with author from email match if possible, otherwise "Unknown user". Send-message functionality disabled for that user.

**Channel renamed in Slack**: detect on next sync and update the linked channel name in Lexnova AI.

**Channel archived in Slack**: surface the linked matter's Slack tab with "This channel is archived. Unarchive in Slack to continue using." No new messages can be sent.

**Privacy on private channels**: the Slack admin chooses whether to grant the app access to private channels. If granted, private channels can be linked. If not, only public channels are usable.

**Notification spam**: bulk operations (e.g., creating 10 tasks at once) should batch into a single notification: "Vincent created 10 tasks on Acme matter".

## Visual references

Slack's own message UI is the reference for message rendering inside the matter Slack tab. The "Connect to workspace" pattern follows Slack's standard OAuth install flow. The matter-channel link pattern resembles Linear's matter-Slack channel binding.

## Accessibility

- Slack message timeline is keyboard navigable (arrow keys move focus between messages)
- Reply input has `aria-label` and supports Enter to send (with Shift+Enter for newline)
- Notifications announce send confirmation for screen readers

## Acceptance criteria

A Lex Nova admin can connect the Slack workspace once. Every matter can be linked to a Slack channel (auto-created or manually linked). Matter detail pages show the linked channel's messages inline. Notifications post to channels on configured events. The assistant uses Slack messages as context. Slack user identities are matched to Lexnova AI users by email.

## Out of scope

- Slash commands inside Slack (e.g., `/lexnova summarize matter` from Slack) — v2 feature
- Direct messages (DMs) integration — only channels, not DMs
- Workflow Builder integration
- Custom emoji and reactions feeding into matter sentiment / status
- Voice / Huddle integration
- Per-user Slack themes / formatting overrides
- Slack Connect (cross-workspace) channels
