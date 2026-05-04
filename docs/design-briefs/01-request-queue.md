# Brief 01 — Concurrent Request Queue

## Context

Lexnova AI's chat assistant currently blocks the user while a request processes. A lawyer who wants to ask three questions in a row must wait for the first answer, then submit the second, then wait again. This is a daily-use blocker. Lawyers think faster than the model and need to fire several questions without waiting between each. Claude.ai handles this gracefully: you submit, the response streams, you can immediately type and submit again, the previous response continues to stream while the new one queues. The queue surface shows "in progress," "queued," and "completed" states across one or many concurrent conversations.

This is the single most-requested UX improvement after the rebrand. It does not require new business logic. It requires a small in-memory queue, a UI affordance to show queue state, and a backend that supports concurrent streaming.

## Audience

Every lawyer using the chat or tabular review surfaces. Particularly relevant for partners doing fast research sweeps and associates running batch DD review.

## Primary user stories

A partner is reviewing a contract. She asks "what is the governing law" and immediately follows with "list any non-compete clauses" without waiting. Both questions are accepted, both responses appear in the chat in submission order, both stream concurrently. She can scroll up and read the first response while the second is still generating.

An associate is on three matters in parallel. He has the assistant open on Matter A in one browser tab, Matter B in another, Matter C in a third. He fires a question on each. All three run concurrently. None blocks any other.

A lawyer accidentally asks the wrong question and wants to cancel. She clicks a small "stop" button on the in-progress message. The stream halts, a partial result is preserved, the queue moves on to the next request.

A lawyer's network drops mid-stream. The queue indicator turns red, an inline retry button appears next to the failed request. She clicks retry; the request resubmits without losing her input.

## Architecture (UI-relevant only)

Each chat session has a client-side request queue (in React state, persisted to sessionStorage so a refresh doesn't lose pending items). Each request has a state: `pending`, `streaming`, `complete`, `cancelled`, `error`. Requests render in submission order in the chat. The "submit" button never blocks; it always accepts new input. The backend supports concurrent streaming responses via Server-Sent Events or Anthropic's streaming API; multiple SSE connections can be open simultaneously per chat.

Concurrency limits: the client allows up to 5 concurrent in-flight requests per chat. Beyond 5, new submissions queue locally and start when an in-flight slot frees. This protects API rate limits and the user from runaway spend.

## Screens

### Chat input area (always-active state)

The input box is never disabled. It accepts text input even while a previous request is streaming. The submit button (`SendHorizontal` icon) is always enabled when the input has text. On submit, the input clears immediately, the new message appears in the message list with `pending` state, and the input is ready for the next message. No blocking spinner.

A small inline status badge appears above the input box when one or more requests are in flight, showing "1 thinking" or "3 thinking" with a soft pulse animation. Clicking the badge opens a popover listing all in-flight requests with their preview text and a "stop" button each.

### Message bubble — request states

Each user message has a paired assistant response slot. The assistant slot's appearance depends on state:

- `pending` (queued, not yet started): light gray background, subtle "Queued..." text in `text-muted-foreground`, no spinner
- `streaming` (response generating): white background, content streams in token-by-token, blinking cursor at the end, small "stop" icon button at the top-right of the bubble (`Square` icon from Lucide)
- `complete`: standard assistant message bubble, no stop button, hover reveals copy and regenerate icons
- `cancelled`: shows partial content (whatever streamed before stop), small italic note below: "Stopped by user"
- `error`: red-tinted bubble, error message inside, "Retry" button at bottom-right

The user message bubble itself never changes state. Only the response bubble does.

### Multi-conversation queue indicator (top of sidebar)

When the user has more than one conversation with active requests, a small indicator appears at the top of the sidebar (just below the Lexnova AI wordmark) with a count: "2 chats running". Clicking it opens a slide-down panel listing each active chat with the chat title, the matter name, and the number of in-flight requests. Each row links to the chat and previews the most recent in-flight request. Useful when a partner has 5 chats open across 3 matters.

### Tabular review queue (in tabular review surface)

The same concept applies to the tabular review's per-cell extraction. Each cell is its own request. When a column is added across 100 documents, 100 cells go into pending state and start processing in batches of 10 concurrent. Each cell shows a tiny spinner while pending. The column header shows progress: "Extracting... 47 / 100". Cells can be retried individually or by column.

### Cancellation

Cancelling an in-flight request: click the `Square` (stop) icon on the streaming bubble. The stream is aborted. The bubble switches to `cancelled` state. The next pending request starts immediately if there is one.

Cancelling all in-flight: keyboard shortcut `Esc` while focused in the chat (when input is empty) cancels all in-flight requests for that chat. A confirmation toast appears: "3 requests cancelled."

## Component additions

- `<RequestQueueBadge>` — the small "X thinking" indicator above the input
- `<RequestQueuePanel>` — the popover listing in-flight requests
- `<MessageBubble>` extended with a `state` prop and the stop button
- `<MultiChatIndicator>` — the sidebar-top indicator when multiple chats have active requests
- `<RetryButton>` — small "retry" affordance for errored requests

Existing `ChatInput` is updated to never disable. Existing `AssistantMessage` is updated with the state prop and stop button.

## Empty / loading / error states

- Empty queue (no in-flight requests): no indicator visible, normal chat
- Single in-flight: badge "1 thinking", subtle pulse
- Multiple in-flight in same chat: badge "X thinking" + popover lists them
- Multiple in-flight across chats: sidebar indicator + chat-level badges
- Error: red-tinted bubble with retry button, error toast
- Network offline: top banner "Offline — requests will resume when you reconnect"; new requests queue with `offline` state until connection restored

## Edge cases

**Tab close while requests in flight**: prompt "You have 2 requests still processing. Close anyway?" — if yes, requests continue server-side and will appear next time the chat is opened (per-chat persistence of message state in DB). If user dismisses, requests continue and tab stays open.

**Backend rate limit hit (429 from Anthropic/Gemini)**: in-flight request fails with a specific error variant. Bubble shows "Rate limit reached. Auto-retrying in 30s..." with a countdown. Retry happens automatically up to 3 times with exponential backoff. After 3 failures, surfaces an error with manual retry button.

**Long-running request (>2 minutes)**: shows extended "still working" indicator. After 5 minutes, prompts "This is taking longer than usual. Cancel?" with cancel/keep waiting choices.

**Conflicting outputs**: if two concurrent requests in the same chat produce contradictory answers (e.g., "what's the deadline" answered twice with different dates), no special handling — both responses appear in order. The lawyer reviews both and decides.

**Tabular review concurrent column adds**: if a user adds two columns at once across many docs, total in-flight cells could be hundreds. Throttle aggressively: max 10 concurrent cells regardless of how many columns/cells are queued. Show overall progress bar.

## Visual references

- Claude.ai's chat behavior is the canonical reference for the input never blocking and responses streaming concurrently
- Existing Mike `AssistantMessage` component is the styling reference for the response bubble — extend it, don't replace it
- Existing `Sonner` toast pattern is the reference for the offline banner and rate-limit notification

## Accessibility

- Streaming responses must announce their final completion via `role="status"` for screen readers (not the streamed tokens themselves, only the "response complete" event)
- Stop button reachable by keyboard (Tab focus on the message bubble brings up the stop button)
- Queue badge announces count on update for screen readers
- Color is never the sole indicator of state — text labels accompany the color in error and cancelled bubbles

## Acceptance criteria

A user can submit 3 messages in quick succession without the second or third being blocked. All three appear in the chat in order, all three stream concurrently. The user can stop any one of them mid-stream. The queue indicator is always accurate. Errors show a retry button. After page refresh, complete and cancelled messages persist; in-flight messages either resume streaming or show a "resume" prompt depending on backend support.

## Out of scope

- Cross-device queue sync (mobile + desktop)
- Server-side queue management (per-user max parallel runs at the API level — that's a backend rate-limit concern)
- Background queue progressing while the tab is closed (only relevant if we add long-running batch jobs later)
- Pause/resume of individual requests (only stop is supported in v1)
