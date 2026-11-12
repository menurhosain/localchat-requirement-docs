# Live Chat Takeover — Implementation Plan (D5)

> **Goal:** A team member watches visitor chats live in the dashboard, clicks **Take over**, and replies in person.
> The visitor sees "Team member joined the chat" (site-wide name from settings) and the team member's replies. While a person is in charge
> **the AI stays silent**. **Hand back to AI** gives the chat back to the bot.
>
> **Builds on:** conversation log (schema v3: `wp_mnaisp_conversations` + `wp_mnaisp_messages`, uuid from the
> widget, `ConversationLogger`, Conversations tab + viewer drawer), public `/chat/ask` (SSE), widget `useChat`
> (conversation in `sessionStorage`).
>
> **Not in this plan:** smart sales alerts (D6, will link to it), a visitor "Talk to a human" button (D2/D3),
> typing indicators, WebSockets, mobile app, email/Telegram notifications.

Created: 2026-11-12 · Status: **draft for review** · Decisions from the user's review: 2026-11-12 (section 11)

---

## 1. Summary

| Item | Decision (proposed) |
|---|---|
| Live transport | **Short polling over REST**, no SSE or WebSockets. Works on shared hosting, which has few PHP workers, and long-held connections would block them. Client code hides the transport behind a small interface so SSE or a push service can be added later |
| Conversation state | New column `conversations.mode` = `ai` \| `human`, plus `agent_id` (the WP user in charge) |
| Message types | New column `messages.kind`: `qa` (today's question + AI answer row), `visitor` (visitor message while a human is in charge), `agent` (team reply), `event` (joined / handed back / visitor left) |
| AI silent | `/chat/ask` checks `mode` first. If `human`: save as `visitor`, return JSON `{handled: "human"}`, no search, no AI call, no cost |
| Who can take over | **Administrators and Editors** (user's call). Capability `edit_others_posts` (admin + editor; WooCommerce shop managers have it too), filter `mnaisp_live_chat_capability`. Editors see only the Live chat tab |
| One agent at a time | (Dashboard shows real WP names.) Atomic conditional `UPDATE`: the first agent wins. Others see "Sarah is handling this chat" and can **force** take over (with a confirm dialog) |
| Visitor online | Widget polls update `conversations.visitor_seen_at`. "Online" = seen within 90 s, "Left" after 3 min (changed 2026-11-12, was 60 s / 2 min) |
| Agent online | Admin polls update `conversations.agent_seen_at`. Used for auto hand-back after **10 min** (user's call) |
| Name shown to visitor | One setting **"Team member name"**, default "Team member" (user's call). Not the WP user name |
| Plan | **Free**, no licence check (user's call) |
| Requires | Conversation recording ON. When recording is off, the Live chat tab shows a notice and the widget does not poll |
| Admin UI | New tab **Live chat** (`#/live`): list of active chats on the left, thread + Take over / reply / Hand back on the right |
| Visitor UI | New bubble type "team member" (name label), centered system notes, placeholder "Message Team member…", unread dot on the launcher |

---

## 2. Flow

```
Visitor widget                      WordPress (REST)                         Dashboard (Live chat tab)
──────────────                      ────────────────                         ─────────────────────────
ask "Do you ship to Canada?" ─────▶ /chat/ask  mode=ai ─▶ AI answers (SSE)
                                    row kind=qa                     ◀──────── poll GET /live/chats (5 s)
poll POST /chat/live (15 s) ──────▶ {mode: ai, messages: []}                  sees chat, opens it
                                                                    ◀──────── POST /live/chats/{id}/takeover
                                    mode=human, agent_id=7,
                                    row kind=event "joined"
poll ─────────────────────────────▶ {mode: human, agent: "Team member",
                                     messages: [event joined]}
"Team member joined the chat" note
poll speeds up to 3 s
ask "My order arrived damaged" ───▶ /chat/ask  mode=human ─▶ NO AI
                                    row kind=visitor,  JSON {handled: human}
                                                                    ◀──────── poll GET /live/chats/{id}/messages (2 s)
                                                                              sees message, types reply
                                                                    ◀──────── POST /live/chats/{id}/reply
                                    row kind=agent
poll ─────────────────────────────▶ messages: [agent reply]
shows "Team member" bubble
                                                                    ◀──────── POST /live/chats/{id}/handback
                                    mode=ai, row kind=event "handed_back"
poll ─────────────────────────────▶ "Chat handed back to the AI assistant"
next question ────────────────────▶ /chat/ask mode=ai ─▶ AI answers again
```

---

## 3. Why polling (and what it costs)

| Option | Shared hosting | Notes |
|---|---|---|
| **Short polling (chosen)** | ✅ | One short request each time. Simple, works behind every cache and proxy. Delay = poll interval |
| SSE / long polling | ⚠️ | Holds a PHP worker per open chat. A host with 5–10 workers is full after a few visitors |
| WebSockets | ❌ | Needs its own server (Node or Pusher-like service). Possible later with the managed gateway (proposal 2) |
| WP Heartbeat API | admin only | Interval 15–60 s, too slow for chat; not loaded on the front end |

**Load control** (each poll = one WP boot ≈ 30–80 ms PHP + one indexed query):
- The widget polls **only after the visitor sent a first message** (the conversation exists on the server) and recording is on.
- Interval: `ai` mode, panel open: **15 s** (only to notice a takeover). `human` mode: **3 s**. Panel closed: 15 s in human mode
  (unread dot), **60 s in ai mode** (changed 2026-11-12: was no polling, visitors looked "left" while on the page). Tab hidden (`document.hidden`): 30 s.
- **Idle back-off (user's call):** no interaction for **10 min** (no message sent or received, no typing/click in the chat) →
  poll every **60 s**. Any interaction or a new agent message → back to the normal interval. Same rule in the dashboard
  (no new message and no agent action for 10 min → list and thread poll every 30 s).
- The server sends the next interval in each response (`poll` ms), so we can tune it without a new widget build.
- `visitor_seen_at` is written at most once every 20 s per conversation (no write on every poll).
- POST (not GET) + `nocache_headers()` so page caches and CDNs never cache it.
- Own rate limit for `/chat/live` (e.g. 40/min per visitor), separate from the question limits.
- Rough load: 20 visitors with chat open, 15 s = ~1.3 req/s; 3 chats taken over at 3 s = +1 req/s. OK for normal hosting.
- Later (if needed): a lighter endpoint that skips full plugin load, or a push service.

---

## 4. Database (schema v4)

`wp_mnaisp_conversations` — new columns:

| Column | Type | Meaning |
|---|---|---|
| `mode` | varchar(10) NOT NULL DEFAULT 'ai' | `ai` or `human` |
| `agent_id` | bigint unsigned NULL | WP user in charge while `human` (kept after hand-back = last agent) |
| `taken_over_at` | datetime NULL | When the current takeover started |
| `visitor_seen_at` | datetime NULL | Last widget poll or question (online dot) |
| `agent_seen_at` | datetime NULL | Last dashboard poll by the agent in charge (auto hand-back) |
| KEY `live` (`visitor_seen_at`) | | For the Live chat list |

`wp_mnaisp_messages` — new columns:

| Column | Type | Meaning |
|---|---|---|
| `kind` | varchar(10) NOT NULL DEFAULT 'qa' | `qa`, `visitor`, `agent`, `event` |
| `user_id` | bigint unsigned NULL | Agent who wrote an `agent` row or caused an `event` |

How each kind uses the existing columns (no new text columns):

| kind | `question` | `answer` | `status` | `reason` |
|---|---|---|---|---|
| `qa` | visitor question | AI answer | pending / answered / dont_know / error (as today) | gate reason |
| `visitor` | visitor message | — | `human` | — |
| `agent` | '' | reply text | `sent` | — |
| `event` | '' | — | `sent` | `joined`, `handed_back`, `auto_handed_back`, `visitor_left` |

Other changes:
- `message_count` counts `qa` + `visitor`. `unanswered_count` and the **Unanswered questions** list stay `kind = 'qa'` only
  (questions a human answered are not "AI didn't know").
- `last_message_at` also updated by `agent` rows.
- Migration: `dbDelta` adds the columns; old rows get the defaults (`ai`, `qa`). `docs.html` updated.
- Purge cron / delete: unchanged (messages deleted with the conversation).

---

## 5. Server (PHP)

### 5.1 `Conversations\LiveChatService` (new)
All live rules in one place; REST controllers stay thin.

| Method | What it does |
|---|---|
| `takeover(int $conversation, int $user, bool $force): Result` | `UPDATE … SET mode='human', agent_id=%d, taken_over_at=NOW() WHERE id=%d AND (mode='ai' OR agent_id=%d OR %d=1)`. 0 rows = someone else holds it, return 409 with their name. Adds `event joined` |
| `handback(int $conversation, int $user, string $reason = 'handed_back')` | Only the agent in charge (or force). `mode='ai'`, adds `event` |
| `reply(int $conversation, int $user, string $text)` | Only the agent in charge, else 409 "Take over first". Text ≤ 2000 chars, `sanitize_textarea_field`. Adds `agent` row |
| `visitor_message(int $conversation, string $text): int` | Called by `/chat/ask` in human mode. Adds `visitor` row |
| `visitor_poll(string $uuid, int $after): array` | Returns `{mode, agent:{name}, messages:[agent + event rows with id > after], poll}`, throttled `visitor_seen_at` update, auto hand-back check |
| `admin_poll(int $conversation, int $user, int $after, string $since): array` | New rows (`id > after`) + rows changed since `since` (a pending `qa` that got its AI answer). Updates `agent_seen_at` when the user is the agent in charge |
| `active(): array` | Chats with `visitor_seen_at` in the last 10 min, or `mode='human'`: uuid-free summary (id, page URL, user, mode, agent, last message preview, counts, online) |
| `auto_handback_if_idle(object $conversation)` | If `mode='human'` and `agent_seen_at` older than N min (Q3), hand back with `auto_handed_back` |

Name shown to visitors: the **Team member name** setting (default "Team member"), filter `mnaisp_live_agent_name`.
The real WP user is stored (`agent_id`, `user_id`) and shown only in the dashboard.

### 5.2 `/chat/ask` change (`ChatController::ask`)
Before rate-limit/search/AI:
1. Find the conversation by uuid. If `mode = 'human'` (after the auto hand-back check): save `visitor` row, update
   `visitor_seen_at`, return `200 JSON {handled: "human", messageId}`. **No search, no AI, no embedding needed.**
2. Otherwise: as today.
- Rate limits still apply in human mode (spam protection), but the "chat not ready" check is skipped (a human can
  answer even when the index is empty).
- If a takeover happens **while an AI answer is streaming**, that answer finishes normally. Only new messages go to the human.

### 5.3 REST routes

Public (widget, no nonce, like `/chat/ask`):

| Route | Body | Returns |
|---|---|---|
| `POST /chat/live` | `{conversationId, after}` | `{mode, agent, messages[], poll}` |

The conversation uuid is the visitor's secret (random UUID v4, 122 bits, never shown on public pages). Unknown uuid
→ `{mode: "ai", messages: [], poll}` (no 404, so uuids can't be probed for existence). Honeypot not needed (no writes
except `visitor_seen_at`).

Admin (`LiveChatController`, cookie + nonce, capability check):

| Route | Body | Returns |
|---|---|---|
| `GET /live/chats` | — | active chats list (polled every 5 s) |
| `GET /live/chats/{id}/messages` | `after`, `since` | new/changed rows + conversation state (polled every 2 s while open) |
| `POST /live/chats/{id}/takeover` | `{force}` | state, or 409 `{agent}` |
| `POST /live/chats/{id}/reply` | `{text}` | the new row |
| `POST /live/chats/{id}/handback` | — | state |

`ConversationsController` `GET /conversations/{id}` also returns `kind`, `user_id` (+ agent name) per message.

### 5.4 Wiring
`Plugin.php` builds `LiveChatService` (repository + settings) and passes it to `ChatController` and the new controller.
`ConversationRepository` gets: `find_by_uuid`, `set_mode` (conditional), `add_message(kind, …)`, `touch_visitor`,
`touch_agent`, `messages_after`, `active`.

---

## 6. Widget (visitor)

Files: `widget/useChat.ts`, `widget/ChatPanel.tsx`, `widget/types.ts`, new `widget/useLive.ts`, `widget/loader.ts`.

- **Entry types**: `ChatEntry.role` adds `'agent'` (text + `agentName`) and `'system'` (centered note). Stored in
  `sessionStorage` with the rest (last 40). Also store `liveAfter` (last seen live row id) and `mode`.
- **`useLive` hook**: the poll loop (`setTimeout`, not `setInterval`; interval from the server's `poll`). Starts after
  the first message. Pauses/slows on `visibilitychange`. Aborts on unmount. Errors: back off (×2, max 60 s), never show an error.
- **Human mode**:
  - Header subtitle / note "You're chatting with Team member" (the site-wide name).
  - Placeholder "Message Team member…".
  - Send: no embedding, no streaming. Visitor bubble appears at once with a small "sent" state; `/chat/ask` returns
    `{handled: "human"}`. No Stop button.
  - Suggested questions hidden.
- **Agent bubble**: bot bubble colours + label = Team member name (CSS class `.mnaisp-agent`, uses existing colour vars).
  Text rendered with the existing small Markdown renderer (links allowed, no HTML).
- **Panel closed + new agent message**: unread dot on the launcher (loader.ts) + launcher label "New message".
- **↺ new chat while human**: new uuid as today. The old conversation gets an `event visitor_left` on the next agent
  poll when the visitor stays offline (no extra request from the widget).
- **History for the AI after hand-back** (user's call: one full conversation): `completedTurns` treats `visitor` →
  `agent` messages as user/assistant pairs (several visitor or agent messages in a row are joined), so the AI knows what
  the team promised. The "Conversation memory" setting (last N messages) still applies.
- **Preview (Style tab)**: shows one sample agent bubble + system note so owners can see the colours.

---

## 7. Dashboard (admin)

New page `pages/LiveChat.tsx` (`#/live`, menu label "Live chat"), components `components/live/ChatList.tsx`,
`components/live/ChatThread.tsx`, api `api/live.ts`.

**Editor access:** the whole admin menu is `manage_options` today (`AdminPage::CAPABILITY`). Change: the menu page
uses `edit_others_posts`; boot data gets `caps: {manage, liveChat}`; the React app shows only the Live chat tab (and
opens it by default) when `manage` is false. All other REST routes keep `manage_options`, so an editor can't reach
settings, content or the log even by calling the API.

**Left: chat list** (poll 5 s, only while the tab is visible)
- Each row: online dot, visitor ("Guest" or WP user name), page title/URL, last message preview + time, badge:
  `AI` / `You` / `Sarah` (other agent), unread count.
- Filter: All / Taken over by me. Sorted by last message.
- Empty state: "No visitors are chatting right now." Recording off → notice with a link to the settings card.

**Right: thread** (poll 2 s while open)
- Same rendering as the Conversations viewer (questions, AI answers with sources, "I don't know" marks), plus agent
  bubbles and event notes. A pending AI answer shows "AI is answering…".
- Header: visitor info, page link, online/offline, mode badge, buttons:
  - `ai` → **Take over** (primary).
  - `human` + me → **Hand back to AI**.
  - `human` + other agent → "Sarah is handling this chat" + **Take over anyway** (confirm dialog).
- Reply box: enabled only when I'm in charge. Enter = send, Shift+Enter = new line, 2000 chars max. Optimistic bubble,
  replaced by the server row.
- Visitor offline for > 3 min while I'm in charge: note "Visitor left the page", Hand back still possible.

**Extras**
- Browser tab title shows `(2) Live chat` when chats I hold have unread visitor messages; short sound (off by default,
  toggle stored in `localStorage`).
- Admin menu bubble count (chats taken over with unread messages) — later, with D6.
- Conversations tab: viewer drawer shows agent/event rows; "Open in Live chat" button when the visitor is online.
- Overview: "Live now: N visitors" (optional).

---

## 8. Security and privacy

- Admin routes: nonce + capability (`mnaisp_live_chat_capability` filter). Agent actions check `agent_id` server-side.
- Visitor cannot create `agent`/`event` rows: `kind` is never taken from the request.
- Agent text: `sanitize_textarea_field`, stored as plain text, rendered by the widget's own renderer (only
  `[text](https://…)` links, `rel="noopener"`, no HTML).
- uuid validated with the existing `UUID_PATTERN`; never printed in the admin list (ids only).
- `/chat/live` rate limited; returns only `agent` and `event` rows (the visitor already has their own messages).
- Privacy-policy text (`wp_add_privacy_policy_content`): add "Team members may read and reply to chats in real time".
- Agent name shown to visitors = data about staff; option to show a generic "Team member" (Q2).
- Retention: same 90-day purge.

---

## 9. Edge cases

| Case | Behaviour |
|---|---|
| Takeover while AI answer is streaming | Answer finishes; next message goes to the human |
| Two agents click Take over at once | Atomic UPDATE: one wins, other gets 409 + name |
| Agent closes the dashboard while in charge | Auto hand-back after N min of no agent poll (Q3), event `auto_handed_back`, visitor sees "The AI assistant is back" |
| Visitor reloads / goes to another page | `sessionStorage` keeps uuid + entries + `liveAfter`; polling resumes; mode comes from the server |
| Visitor opens a new tab | New, empty conversation (per tab, user's call). Old tab keeps its chat |
| Visitor closes the tab | Conversation is gone for the visitor. Agent sees "Visitor left" after 3 min offline; replies after that are stored but never seen |
| Visitor duplicates the tab (Chrome "Duplicate") | Browser copies sessionStorage: both tabs share one conversation and both show replies. Harmless |
| Visitor message sent while mode changes | `/chat/ask` reads mode at request time; whichever state is current wins |
| Recording turned off during a takeover | Open takeovers handed back to the AI; live endpoints return `mode: ai` (decision A) |
| Old cached widget JS (no live support) | It never polls, so it never sees a takeover or agent replies. The JSON for human mode also carries the answer fields the old widget reads, with the text "A team member will reply shortly.", so it doesn't show an error. Fixed when the page cache refreshes |
| Page cache | Widget calls are POST + no-cache headers |
| Visitor rate limit reached in human mode | Same 429 message as today |

---

## 10. Milestones

Each milestone = one commit, testable on the dev site.

| # | Milestone | Done when |
|---|---|---|
| L0 | Schema v4 + repository methods (`kind`, `mode`, agent fields), Unanswered/totals keep `qa` only, `docs.html` updated | Migration runs, old data intact, existing tabs unchanged |
| L1 | `LiveChatService` + admin REST (`/live/chats…`) + `/chat/ask` human branch + public `/chat/live` | Takeover/reply/handback via WP-CLI `rest_do_request`; `/chat/ask` returns `handled: human` with no AI call |
| L2 | Widget: `useLive` polling, agent/system entries, human-mode send, unread dot, history pairs | Takeover from WP-CLI shows up in the widget; replies appear within 3 s |
| L3 | Admin **Live chat** tab: list, thread, Take over / reply / Hand back, force takeover | Full flow in two browsers (visitor + admin) |
| L4 | Conversations viewer shows agent/event rows + "Open in Live chat"; privacy text; Style preview sample; auto hand-back | Old + new chats display correctly |
| L5 | Polish: tab-title count, sound toggle, poll tuning, `docs/live-takeover.md` study doc, README | Manual test script below passes |

Tests: deferred (user's call) — manual script only for now:
1. Visitor asks 2 questions (AI answers). Admin sees the chat in Live chat within 5 s.
2. Admin takes over → visitor sees joined note within 15 s (panel open).
3. Visitor writes → no AI call (check Conversations: `kind=visitor`, no tokens) → admin sees it within 2 s.
4. Admin replies → visitor sees bubble within 3 s, with name.
5. Second admin tries Take over → sees "Sarah is handling", force works.
6. Hand back → visitor sees note, next question answered by AI with the human part in its history.
7. Visitor closes the panel → admin replies → unread dot on launcher.
8. Admin closes the dashboard → auto hand-back after N min.

---

## 10b. Built differently from this plan (L1, 2026-11-12)

- **"Visitor left" is not stored as an event row.** The dashboard gets `visitorOnline` (seen < 90 s) and `visitorLeft`
  (seen > 3 min) flags computed from `visitor_seen_at`. No duplicate rows, nothing to clean up. `EVENT_VISITOR_LEFT`
  stays reserved.
- **Rate limits in human mode:** only the per-minute limit applies. The per-day and site-wide limits protect the AI
  budget, and a person answering costs nothing.
- **Admin thread poll** returns `serverTime`; the next poll sends it back as `since` (no browser clock involved).
- **Questions also mark the visitor online** (`visitor_seen_at`), so a chat shows in the Live chat list right after
  the first question, before the widget's first poll.
- **Team member name** lives in the conversation settings (`team_name`, REST `teamName`); the settings UI comes with L3.

## 10c. Built differently from this plan (L2–L4, 2026-11-12)

- **Team member replies are plain text** in the widget (line breaks kept, no Markdown, no links). Safer; links can
  come later.
- **Widget after a page change:** the loader reads the stored live mode; when a team member is in charge it loads the
  chat closed right away, so polling and the unread count continue without a click.
- **Loader must not import widget modules** (it would load the loader file a second time by another URL and draw two
  buttons). The stored-mode check is copied into loader.ts.
- **Editors** get the plugin menu with only the Live chat tab (`AdminPage::menu_capability()`); all other REST routes
  still need `manage_options`.
- **Dashboard idle back-off:** 10 min without a new message or action → list and thread poll every 30 s.
- **Shared `MessageRow`** renders every message kind in both the Live chat thread and the Conversations viewer.
- **Style preview** is now 600 px high (real widget height) and scrolls, with a sample team member reply.
- Not built yet (L5): tab-title unread count, sound, study doc, README.

## 10d. Online status fix (2026-11-12, user's call)

Two weak spots found after L4: (1) with the AI in charge the closed widget didn't poll, and after a page change the
chat wasn't loaded at all, so a visitor still on the page showed "Left" after ~2 min; (2) idle polling every 60 s sat
exactly on the 60 s Online limit, so the dot flickered. Fix: the chat loads closed on every page once the tab has
chatted and checks in every 60 s also in AI mode; Online = seen within 90 s, Left = over 3 min. Every poll pace is
≤ 60 s, so a visitor on the page never drops below Online. Cost: one small request per minute per visitor who chatted.

## 11. Decisions (user's review, 2026-11-12)

| # | Question | Decision |
|---|---|---|
| 1 | Who can take over | Administrators + Editors |
| 2 | Name shown to visitor | Setting "Team member name", default "Team member" |
| 3 | Auto hand-back | Yes, after 10 min without the agent's dashboard polling |
| 4 | AI history after hand-back | Yes, one full conversation (visitor ↔ team messages included) |
| 6 | Conversation scope | Per tab (sessionStorage), as today |
| 7 | Visitor left, reply waiting | No email. Tab closed = conversation gone for the visitor; coming back = new conversation |
| 8 | Free or Pro | Free, no licence check |
| 9 | Polling | Intervals OK + idle back-off: no interaction for 10 min → 60 s (widget), 30 s (dashboard) |

| A | Recording off | Live chat works **only when recording is on**. Recording off: widget never polls, `/chat/live` returns `mode: ai`, Live chat tab shows a notice with a link to the recording setting. Turning recording off during a takeover hands open chats back to the AI |
| B | Team member name | **One name for the whole site** (setting "Team member name", default "Team member"). Every agent shows under it; the real WP user is visible only in the dashboard |

All questions answered. Next: L0.
