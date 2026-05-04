# Brief 06 — Gmail Integration

## Context

Lex Nova lives in email. Most client communication, opposing counsel correspondence, internal coordination, and document handoffs happen via Gmail. Today, this email context lives entirely outside Lexnova AI. A lawyer reading the assistant chat about a matter has no awareness of the 47 emails about that same matter; the AI has no awareness either. A partner asking "what is the status" has to ask the assistant AND check email separately.

This brief integrates Gmail at the matter level. Each lawyer connects their own Gmail account via OAuth (firm-wide consent isn't possible without a Workspace admin app). Threads can be linked to matters automatically (by client domain) or manually. The assistant can read linked threads as context. The matter detail page surfaces an Email tab. A "needs reply" flag drives a daily inbox view of matters waiting on Lex Nova.

This is the highest-impact integration. It is also the one with the longest lead time because Google must verify the OAuth app for sensitive scopes (typically 5-7 business days, sometimes longer).

## Audience

Every Lex Nova lawyer who handles client email. The most active beneficiaries are partners whose volume is highest and whose memory of "where is matter X" matters most.

## Primary user stories

A partner connects her Gmail to Lexnova AI. After consent, the system fetches her last 30 days of email and auto-categorizes by client domain. Acme Corp emails go into the Acme matter, Beta emails into the Beta matter, and so on. Unmatched emails go to an Inbox view where she can manually link them.

An associate opens the Acme matter's Email tab. He sees the full thread history, latest message at top. The thread reads exactly like Gmail. He can compose a reply directly inside Lexnova AI; it sends from his Gmail and appears in his sent folder.

A lawyer asks the assistant "What did Acme reply about the deposit?" The assistant searches the linked Gmail threads for that matter, finds the relevant message, quotes it in the answer with a link to the full thread.

A partner opens her morning dashboard. She sees "5 matters awaiting client reply for more than 3 days". She clicks each, glances at the last message, decides to nudge the client or escalate.

A lawyer marks a thread as "Needs reply by me" — the matter shows up in her My Tasks-like surface with the thread as the prompt for action.

## Architecture

OAuth scopes requested: `gmail.readonly` (read messages and threads), `gmail.send` (send replies), `gmail.modify` (apply labels — used to mark Lexnova-tracked threads). All three require Google's verification process. Submit the app for verification on Day 1 of this sprint; build the integration UI while waiting.

Each Lexnova user OAuth's their own Gmail account. The firm cannot share one OAuth — Google's per-user consent is mandatory.

Once connected, a background sync polls Gmail every 5 minutes for new messages. Webhook (push notifications via Pub/Sub) added later for instant updates.

Threads are linked to matters by:

- Manual link (most reliable) — user opens a thread and clicks "Link to matter"
- Domain match (heuristic) — emails to/from a domain that's been linked to a matter auto-link
- Subject keyword match (weakest) — fallback for ambiguous threads, surfaces as "suggested link" with confirm

## Schema delta

New tables:

- `gmail_connections` — one per Lexnova user: oauth tokens, refresh tokens, last sync time
- `gmail_threads` — cached thread metadata: thread_id, subject, snippet, last_message_at, participants jsonb, labels jsonb
- `gmail_messages` — cached message bodies (or just metadata + on-demand fetch for body)
- `matter_email_links` — many-to-many between matters and gmail threads, with link_type (manual, domain, suggested), link_user_id (who linked)
- `matter_email_domains` — for auto-linking: matter_id, domain (e.g., "acme.com")

## Screens

### Connect Gmail (in profile / settings)

In the user profile (Brief 09), a "Connections" section. "Gmail" row shows a Connect button if not connected, or "Connected as vincent@lexnovapartners.com" with a Disconnect option if connected. Connect triggers Google OAuth flow in a popup.

After consent, a small status indicator: "Syncing initial 30 days... 47% complete (1,247 of 2,650 threads)". When done, "Last sync 2 minutes ago".

### Email tab on matter detail

Add an "Email" tab to the matter detail page (after Tasks, before Assistant). Shows linked threads sorted by most recent activity:

- Each thread row: sender names, subject, snippet of the latest message, timestamp, participant avatars, message count badge, "Needs reply" tag if applicable
- Hover a row: "Open thread" / "Unlink" / "Mark needs reply" actions
- Click a row: opens the thread in a side panel showing all messages in chronological order, latest at the bottom

Thread view (side panel):

- Each message: sender, recipients, date/time, body (rendered HTML if Gmail had HTML, otherwise plain text)
- Inline attachments rendered as cards with download links
- A reply composer at the bottom: To/Cc/Bcc fields, subject (auto-populated), rich-text body
- Send sends via Gmail API as the connected user; the message appears in the user's Gmail Sent folder
- Footer actions: "Mark as needs reply" / "Mark as resolved" / "Unlink from matter"

### Inbox view (`/inbox`)

A new top-level route /inbox showing all threads across all of the user's matters, grouped by:

- Needs reply (red header) — threads marked needs-reply or awaiting our response
- Awaiting client (amber) — threads where the last message was sent by Lex Nova more than 3 days ago
- Recent activity (default) — threads with new messages in the last 24 hours

Each thread row shows the matter name, sender, subject, last activity. Click navigates to the thread in its matter.

### Suggested links

When a new email arrives that doesn't auto-link by domain but seems matter-related (e.g., subject contains the matter name or known counsel email), surface as a "Suggested link" toast with options Confirm / Dismiss / Link to different matter.

### Linking a thread manually

In the user's Gmail (separate UI in Lexnova AI Inbox view, not in actual Gmail.com): each thread has a "Link to matter" button that opens a matter picker (search and select). Selecting a matter creates the link and the thread disappears from "unlinked" view.

Domain auto-link can also be enabled per matter: in the matter settings, a "Auto-link emails from these domains" field allows adding domains. Future emails matching those domains auto-link.

### Compose new email from a matter

From the matter detail page, a "Send email" button (next to the existing matter action buttons) opens a compose modal pre-filled with:

- To: matter participants (if known)
- Subject: matter name + reference (configurable in profile)
- Body: empty, with optional template insertion

Sending creates a new thread and links it to the matter immediately.

### Privacy and inbox-wide search

The lawyer's full Gmail data is synced. To respect privacy, only emails linked to a matter are visible to other members of the firm (with shared_with access). All other emails remain private to the connecting user.

The assistant only reads threads explicitly linked to the current matter. It does not silently search the user's entire Gmail.

## Component additions

- `<GmailConnectionStatus>` — the connect/disconnect card in profile
- `<EmailThreadList>` — list of linked threads on the matter Email tab
- `<EmailThreadView>` — side panel with messages and reply composer
- `<InboxPage>` — the `/inbox` route
- `<MatterEmailLinker>` — popover for linking a thread to a matter
- `<EmailComposer>` — rich-text composer for new emails and replies
- `<NeedsReplyBadge>` — small chip for threads awaiting reply

## Empty states

- Email tab on a matter with no linked threads: "No emails linked to this matter yet. Emails from configured domains will auto-link, or use 'Link from inbox' to attach existing threads." Primary CTA: "Configure auto-link domains"
- Inbox with no threads: "Connect Gmail to see your matter-relevant inbox here." with Connect Gmail CTA
- Inbox after connect, all threads triaged: "Inbox zero. Nice." with no CTA

## Edge cases

**OAuth token expired or revoked**: surface a banner at the top of the matter Email tab "Gmail disconnected. Reconnect to keep emails in sync." with a Reconnect button.

**Email has attachments larger than R2 quota**: store metadata only, link out to Gmail for the actual attachment download.

**Outgoing email with confidential content**: lawyers may not realize Lexnova AI is sending via the Gmail API. Send confirmations show "Sent from your Gmail account" prominently to avoid confusion.

**Spam filter false positives**: auto-linked emails should be filtered to not include emails Gmail marked as spam (use Gmail's own spam label).

**Quoted text and signatures**: when displaying a thread, collapse quoted previous emails by default (show "Show 4 quoted messages") — same as Gmail.

**Replying to all vs. replying**: composer defaults to Reply (only original sender). Reply All is a one-click option.

**Compliance retention**: do not delete cached email bodies even if user disconnects Gmail. Lex Nova may have ethical / regulatory obligations to retain client correspondence. Disconnecting just stops new sync.

**Verification delay from Google**: while waiting for OAuth verification, the app shows a banner "Pending Google review" and only the partner with developer access can use the integration. After verification, opens to all users.

## Visual references

Gmail's own thread view is the reference for message rendering. Front (the email-shared inbox tool) is the reference for the team-shared / linked-to-matter pattern. Superhuman is the reference for keyboard speed (left/right arrows to navigate threads, J/K up/down).

## Accessibility

- Email composer rich-text controls are keyboard-only navigable
- Thread view announces new messages arriving for screen readers
- Suggested-link toasts are dismissible with keyboard

## Acceptance criteria

A lawyer can connect Gmail, see linked threads on each matter, read messages, and reply directly from Lexnova AI. Auto-link by domain works. Manual link works. The /inbox route shows the user's matter-relevant threads grouped by status. Replies sent from Lexnova AI appear in Gmail Sent. The assistant can quote linked emails as context. Unconnected users do not see anyone else's email.

## Out of scope

- Calendar integration (separate brief, much later)
- Email templates (Brief 09 covers per-user templates, including email templates)
- Email scheduling (send later)
- Email tracking (read receipts, click tracking)
- Sub-folders / nested labels
- Gmail filters / rules creation from Lexnova AI
- Forwarding rules
- Vacation auto-responder
- IMAP support for non-Gmail users (Outlook is potential v2)
