# Chatbot Analytics — Requirements

Status: requirements only (2026-11-17). Architecture and implementation plan come later.
Visual version: `docs/analytics-requirements.html`.

## 1. Goal

Show the site owner how visitors use the chatbot, step by step:

1. How many **unique visitors** land on a page where the chatbot is present.
2. How many of them **open** the chatbot.
3. How many **start a conversation**.
4. How many **talk to a real person** (team member via Live chat).
5. How much **time** they spend with the chatbot.
6. **On which page** each visitor opened the chatbot for the first time.

The owner should be able to answer: "Is the chatbot used? Where do people open it? How many
end up needing a human? Is it worth keeping on these pages?"

## 2. Scope: which pages count

- Only pages where the chatbot widget loads (floating button or `[mnaisp_chat]` shortcode).
  The tracking code ships inside the widget loader, so pages without the widget are never counted.
- This is **not** a general site analytics tool. A page without the widget is invisible to it.
- Counting must work when the site uses a page cache (WP Rocket, LiteSpeed Cache, Cloudflare, etc.),
  so counting has to happen from the visitor's browser, not from PHP page rendering.

## 3. Definitions

Each metric needs one exact meaning so numbers never contradict each other.

| Term | Meaning |
|------|---------|
| **Unique visitor** | One browser on one device, counted once **per day** however many pages it visits (5 pages = 1 visitor). The same browser on another day counts again (Monday + Tuesday = 2). A range total is the sum of the daily counts. Anonymous: no name, no IP, no fingerprinting. Page views are not counted. |
| **Opened chatbot** | The visitor opened the chat panel themselves (clicked the button or launcher label). Automatic re-opening (e.g. a page reload during a live chat) is not a new open. Shortcode chat (always visible): first click or typing inside it; being on screen is not an open. |
| **Started conversation** | The visitor sent at least one message: typed question, suggested-question click, or message to the team. |
| **Asked for a person** | The chat recorded a request for a human (existing `requested` / `requested_offline` events). |
| **Talked to a person** | A team member sent at least one reply in that chat (taken over + agent message). Taking over without replying does not count. |
| **Time on chatbot** | Time the chat panel was open **and** the browser tab was visible **and** the visitor not idle (60 s without activity). Closed panel, hidden tab, or idle time does not count. |
| **First-open page** | The page URL where the visitor opened the chatbot for the first time **that day**. |

## 4. Required metrics (P0)

### 4.1 Funnel

All steps are counted per day, like unique visitors: "how many came today, opened the chat, started a
conversation, talked to a person". A range (e.g. last 7 days) adds up the days.

Four steps, each with count and % of the previous step and % of the first step:

```
Unique visitors on widget pages     1,000   100%
  └ Opened chatbot                    180    18%
      └ Started conversation          120    67% of opened (12% of visitors)
          └ Talked to a person         15    13% of started (1.5% of visitors)
```

"Asked for a person" is shown next to "Talked to a person" so the owner sees missed requests
(asked but nobody replied).

### 4.2 Time on chatbot

Idle rule (60 s): every activity restarts a 60 s countdown; when it runs out, time stops counting
until the next activity. Activity = typing in the input, clicking in the chat (send, suggestion, link,
button), scrolling the messages, mouse move / touch inside the chat, or a new message arriving (AI
answer or team reply, the visitor is probably reading it).

```
0:00  opens chat, types question      counting
0:20  AI answer arrives               counting (countdown restarts)
1:20  nothing for 60 s                STOP
      away 40 min                     not counted
41:00 comes back, types               counting
41:30 closes panel                    stop
Counted: 1:20 + 0:30 = 1 min 50 s     (not 41 min 30 s)
```


- Average and median time per visitor per day (visitors who opened the chatbot).
- Same, split into: AI-only chats vs chats that talked to a person.
- Total time on chatbot in the range.

### 4.3 First-open pages

- Table: page (title + URL), number of visitors who opened the chatbot there first,
  share of first opens (this page's first opens ÷ all first opens in the range, e.g. 62 ÷ 180 = 34%:
  where chatbot use starts).
- Also per page: unique visitors on that page, so the owner sees the open rate per page
  (opens ÷ unique visitors on that page, e.g. 62 ÷ 240 = 26%).
- Sorted by first opens, top 20, with "show all".
- Clicking a page shows its own funnel: unique visitors on that page → opened on that page → started on that page
  (first message of the day sent there) → asked / talked to a person (credited to that same page). Each visitor who
  started counts on one page only (user's call 2026-11-17).

### 4.4 Date range

- Presets: Today, Last 7 days, Last 30 days, Last 90 days; custom range.
- Site time zone (WordPress setting).
- Daily trend chart for the funnel steps in the range.

## 5. Nice to have (P1, later)

- Compare with the previous period (e.g. +12% opens vs last 30 days).
- Split by device: desktop / mobile / tablet.
- CSV export of the daily numbers and the pages table.
- Funnel per page as a separate screen.
- Link from a number to the matching conversations in the Conversations tab.
- Join with existing data: answered vs "I don't know" rate, top unanswered questions
  (commercial-requirements E3).

## 6. Who and what is excluded

- Logged-in team members who can use Live chat (capability `edit_others_posts`) are not counted
  by default (setting to include them).
- Admin preview (Style tab preview) and Playground are never counted.
- Known bots and crawlers are not counted.
- Do Not Track / Global Privacy Control: ignored, those visitors are counted (user's call).

## 7. Privacy

- No IP address stored (same rule as conversation log).
- No message text in analytics; it only stores that something happened (open, start, etc.).
- Anonymous random visitor ID only; it cannot be linked to a person unless the visitor is logged in
  and chats (the conversation already stores the user ID).
- Page URLs stored without query strings by default (they can contain emails, tokens, order IDs).
- Retention setting for detail rows (one row per visitor per day): default 90 days, owner can change
  the days or turn deletion off (keep forever). Daily totals (per day, per page per day) are kept forever.
- Deletion runs as a daily WP-Cron job, same as `mnaisp_purge_conversations`: deletes detail rows older
  than the limit in batches (so a big delete never locks the table for long). Daily totals are built
  before detail rows are deleted, so reports for old days keep working. WP-Cron runs on site visits;
  sites with a real server cron (`DISABLE_WP_CRON`) work the same.
- On/off setting for analytics, separate from conversation recording.
- Privacy-policy text added via `wp_add_privacy_policy_content`, like the conversation log.
- Counts without waiting for cookie consent (anonymous one-day ID, user's call). The analytics switch
  is the owner's way out if their law needs consent; the privacy-policy text says what is stored.

## 8. Quality requirements

- No slowdown: tracking must not block page load or the chat; small requests, sent in batches or
  with `sendBeacon` when the page closes.
- Tracking failures are silent and never break the widget.
- Bounded database growth on busy sites (e.g. old detail rows rolled up into daily totals).
- Numbers in the funnel always add up: each step ≤ the previous step.
- Admin screen loads fast for 90 days on a busy site (target < 1 s).
- New admin tab visible to admins only (editors keep seeing only Live chat).
- Works on the WWC branch too (user-facing plugin name only differs there).

## 9. Open questions (all answered 2026-11-17)

1. ~~**Unique visitor across days**~~ ANSWERED 2026-11-17: unique **per day**. Same browser on
   Monday and Tuesday = 2. Range totals = sum of days. A daily anonymous ID is enough (no lasting ID).
2. ~~**Consent**~~ ANSWERED 2026-11-17: count anyway, no waiting for cookie consent (ID is anonymous,
   lasts one day). Site owners who need consent can turn analytics off.
3. ~~**Shortcode chat**~~ ANSWERED 2026-11-17: counts as opened on the visitor's first click or typing
   inside it. Scrolling it into view does not count.
4. ~~**First-open page**~~ ANSWERED by question 1: first open of the day.
5. ~~**Idle time**~~ ANSWERED 2026-11-17: 60 s (see 4.2).
6. ~~**Talked to a person**~~ ANSWERED 2026-11-17: only when a team member sent at least one reply.
   Take-over without a reply does not count.
7. ~~**Exclude team members**~~ ANSWERED 2026-11-17: excluded (not counted) by default; setting can include them.
8. ~~**Retention**~~ ANSWERED 2026-11-17: detail rows deleted after 90 days by default; setting to change
   the days or turn deletion off (keep forever, fine for small sites: 200 visitors/day ≈ 73,000 rows/year).
   Daily totals kept forever. Deletion = daily WP-Cron job, like conversations (see section 7).
9. ~~**Free vs Pro**~~ ANSWERED 2026-11-17: all analytics FREE (P0 and P1 extras).
10. ~~**Recording off**~~ ANSWERED 2026-11-17: analytics keeps counting (own switch). "Asked for a person"
    and "Talked to a person" simply show 0 (live chat needs recording). No extra note.
11. ~~**Do Not Track / Global Privacy Control**~~ ANSWERED 2026-11-17: ignore both, count everyone
    (`navigator.globalPrivacyControl` / `Sec-GPC` and `navigator.doNotTrack` / `DNT` not checked).

All open questions answered. Next = architecture + implementation plan.

## 10. Not in scope

- General web analytics (referrers, countries, bounce rate, pages without the widget).
- Sending events to Google Analytics / GA4 (possible later as an option).
- Per-visitor tracking timeline or session recordings.
- A/B tests of widget texts.
