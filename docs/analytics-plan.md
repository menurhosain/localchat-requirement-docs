# Chatbot Analytics — Implementation Plan

> **Goal:** build the analytics described in `docs/analytics-requirements.md`: a daily funnel (unique visitors on
> widget pages → opened → started → talked to a person, plus asked for a person), time on chatbot (60 s idle rule),
> first-open pages with per-page funnels, date ranges, a daily trend, retention with daily totals kept forever.
>
> **Builds on:** widget loader (`assets/src/widget/loader.ts`, inline config `window.mnaispChat`), `useChat`
> (`/chat/ask`, `sendToTeam`), `LiveChatService` (`request_person`, `reply`), `ConversationLogger::visitor_user_id()`
> (login cookie without nonce), `RateLimiter::allow()` buckets, daily WP-Cron (`mnaisp_purge_conversations`),
> `Migrations` (schema v4), React admin app tabs (`App.tsx`).
>
> **Not in this plan:** P1 extras (previous-period compare, device split screen, CSV export, separate per-page screen,
> links into Conversations, answered / "I don't know" rates). The schema leaves room for them.

Created: 2026-11-17 · Status: **built 2026-11-17 (A0–A5), see section 13** (all questions answered 2026-11-17, section 12)

---

## 1. Summary

| Item | Decision (proposed) |
|---|---|
| Where counting happens | In the **browser** (widget loader), so cached pages count. Server-side for facts the server already knows: started (a message reached `/chat/ask`), asked for a person, team replied |
| Visitor ID | Random 22-char ID in **localStorage**, replaced when the site day changes (server tells the day). No IP, no fingerprint, no user ID stored |
| "Per day" | Day = **site time zone** (`wp_date('Y-m-d')`) when the server receives the event. Unique key `(day, visitor ID)`, so per-day uniqueness holds even if an ID lived across midnight |
| Storage | 3 new tables: `analytics_visits` (1 row per visitor per day), `analytics_pages` (1 row per visitor per page per day), `analytics_daily` (totals per day, and per page per day) |
| Link chat → visit | New column `conversations.visit_id` (last visit that wrote in the chat). Agent reply / person request mark that visit |
| Requests | New public endpoint `POST /mnaisp/v1/analytics/track` with batched events. The browser sends a page only the first time per day; open once per page per day; time on close / tab hidden / every 2 min while active. `sendBeacon` when the page goes away |
| Time on chatbot | Measured in the chat bundle (only loaded when the chat is used): panel open + tab visible + activity within 60 s. Seconds added to the visit row, capped by the server |
| Median | Daily totals keep a **time histogram** (fixed buckets), so median works for any range, also after detail rows are deleted |
| Reports | Finished days from `analytics_daily`; today (and days not yet totalled) computed live from detail rows |
| Rollup + purge | Daily WP-Cron `mnaisp_analytics_daily`: (re)build totals for recent days, then delete old detail rows in 5,000-row batches with a 20 s budget and a follow-up single event |
| Settings | Option `mnaisp_analytics_settings`: `enabled` (default on), `retention_days` (default 90, 0 = keep forever), `include_team` (default off) |
| Admin UI | New tab **Analytics** (`#/analytics`), admins only: date range, funnel, time cards, daily trend, first-open pages table, per-page funnel. Settings: card `AnalyticsSettingsCard` on the **Settings** tab |
| Plan | Free, no licence check |
| Schema | v5 (the planned messages-table rework moves to v6) |

---

## 2. Flow

```
Browser (loader.ts, every widget page)            WordPress                               Tables
──────────────────────────────────────            ─────────                               ──────
page loads, idle
  page not yet sent today? ─ track {view} ──────▶ /analytics/track
                                                  skip: bot UA, team member (setting),
                                                        analytics off
                                                  day = wp_date('Y-m-d')        ───────▶ visits  (day, vid) upsert
                                                                                ───────▶ pages   (visit, page) upsert
                                        ◀──────── {day}   (new day? → new vid next time)

clicks button (or first click/typing in shortcode)
  ─ track {open, page} ─────────────────────────▶                               ───────▶ visits.opened_at, first_open_page
                                                                                         (only if empty: first open of day)
                                                                                ───────▶ pages.opened = 1

types question ─ /chat/ask {…, visitorId} ──────▶ ChatController::ask
                                                  mark visit started            ───────▶ visits.started = 1
                                                  conversation.visit_id = visit ───────▶ conversations.visit_id
                                                  "I want a person" → request   ───────▶ visits.asked = 1

                                                  Live chat: team member replies
                                                  LiveChatService::reply        ───────▶ visits.talked = 1 (via visit_id)

panel closed / tab hidden / every 2 min
  ─ track {time: 37} (sendBeacon) ──────────────▶ += seconds (capped)           ───────▶ visits.active_seconds

                                                  WP-Cron daily:
                                                  rollup recent days            ───────▶ daily (site + per page)
                                                  purge details > retention     ───────▶ delete visits/pages in batches
```

---

## 3. Visitor ID and the day

- **Format:** 16 random bytes, base64url = 22 chars (`crypto.getRandomValues`). The server only accepts `^[A-Za-z0-9_-]{22}$`.
- **Stored** in `localStorage['mnaisp-analytics']`:
  `{ vid, day, sent: [pageKey…], opened: [pageKey…] }`. `sent` / `opened` hold the pages already reported today, so the
  browser skips repeat requests. Shared by all tabs, which makes "unique per day" hold across tabs.
- **Day:** the server is the clock. Each track response returns `{ day }` (site time zone). When it differs from the
  stored day, the browser makes a **new ID** and empties `sent` / `opened`. The visitor's own clock and time zone
  never matter.
- **Why the key is `(day, vid)` too:** an ID created at 23:59 sends its first event at 00:01 → it's a visitor of
  the new day. Correct either way.
- **localStorage blocked** (private mode in some browsers, strict settings): ID kept in memory for this page only, so that
  visitor may count again on the next page. Rare; accepted.
- **Site time zone changed later:** stored days stay as they were. Noted, not handled.

---

## 4. Database (schema v5)

### 4.1 `wp_mnaisp_analytics_visits` — one row per visitor per day

```sql
id               bigint unsigned AUTO_INCREMENT
day              date            NOT NULL
vid              char(22)        NOT NULL
device           varchar(10)     NOT NULL DEFAULT ''   -- desktop|mobile|tablet (stored now, used by P1)
opened_at        datetime        NULL                  -- first open of the day (UTC)
first_open_page  bigint unsigned NULL                  -- analytics_pages.id where it first opened
first_msg_page   bigint unsigned NULL                  -- analytics_pages.id where it sent its first message today
started          tinyint(1)      NOT NULL DEFAULT 0
asked            tinyint(1)      NOT NULL DEFAULT 0
talked           tinyint(1)      NOT NULL DEFAULT 0
active_seconds   int unsigned    NOT NULL DEFAULT 0
created_at       datetime        NOT NULL
updated_at       datetime        NOT NULL
PRIMARY KEY (id), UNIQUE KEY day_vid (day, vid), KEY day (day, opened_at)
```

### 4.2 `wp_mnaisp_analytics_pages` — one row per visitor per page per day

```sql
id         bigint unsigned AUTO_INCREMENT
visit_id   bigint unsigned NOT NULL
day        date            NOT NULL          -- copy of the visit's day, for range queries without a join
page_key   char(16)        NOT NULL          -- first 16 hex of sha1(normalized path)
path       varchar(500)    NOT NULL          -- "/pricing/" (no domain, no query, no #)
title      varchar(200)    NOT NULL DEFAULT ''
opened     tinyint(1)      NOT NULL DEFAULT 0
PRIMARY KEY (id), UNIQUE KEY visit_page (visit_id, page_key), KEY day_page (day, page_key)
```

### 4.3 `wp_mnaisp_analytics_daily` — totals, kept forever

```sql
day              date         NOT NULL
page_key         char(16)     NOT NULL DEFAULT ''   -- '' = whole site
path             varchar(500) NOT NULL DEFAULT ''
title            varchar(200) NOT NULL DEFAULT ''
visitors         int unsigned NOT NULL DEFAULT 0
opened           int unsigned NOT NULL DEFAULT 0    -- site row: visitors who opened; page row: opened on this page
first_opens      int unsigned NOT NULL DEFAULT 0    -- page row only
started          int unsigned NOT NULL DEFAULT 0    -- page row: first message of the day sent on this page
asked            int unsigned NOT NULL DEFAULT 0    -- page row: those visitors who asked for a person
talked           int unsigned NOT NULL DEFAULT 0    -- page row: those visitors who talked to a person
seconds_total    bigint unsigned NOT NULL DEFAULT 0
seconds_ai       bigint unsigned NOT NULL DEFAULT 0 -- openers who did not talk to a person
seconds_person   bigint unsigned NOT NULL DEFAULT 0
openers_ai       int unsigned NOT NULL DEFAULT 0
openers_person   int unsigned NOT NULL DEFAULT 0
time_buckets     varchar(255) NOT NULL DEFAULT ''   -- JSON {all:[…], ai:[…], person:[…]} counts per bucket
devices          varchar(255) NOT NULL DEFAULT ''   -- site row only: JSON {desktop:[visitors,opened,started,talked], mobile:[…], tablet:[…]}
updated_at       datetime     NOT NULL
PRIMARY KEY (day, page_key)
```

**Time buckets** (seconds, upper bounds): 10, 30, 60, 120, 180, 300, 600, 900, 1800, 3600, ∞ (11 buckets).
Median = bucket that holds the middle visitor, shown as a value interpolated inside it ("≈ 1 m 20 s"). Same method for
today (computed live) and old ranges, so numbers never jump when details are deleted. Average stays exact
(`seconds / openers`).

### 4.4 `wp_mnaisp_conversations` — one new column

`visit_id bigint unsigned NULL` — the visit that last wrote in this chat. Set on every visitor message, so a chat
continued on the next day credits the new day's visit.

### Row counts (example: 5,000 visitors/day, 3 widget pages each)

| Table | Per day | 90 days |
|---|---|---|
| visits | 5,000 | 450,000 |
| pages | 15,000 | 1.35 M |
| daily | 1 + pages seen (≈ 200) | forever, ≈ 75,000/year |

Small site (200/day, retention off): ≈ 73,000 visits + ≈ 220,000 pages rows per year. Fine for MySQL with these indexes.

---

## 5. Server

### 5.1 New classes (`includes/Analytics/`)

| Class | Job |
|---|---|
| `AnalyticsSettingsService` | Option `mnaisp_analytics_settings` {enabled, retention_days, include_team}; validate/save like `ConversationSettingsService` |
| `AnalyticsRepository` | Upserts (`INSERT … ON DUPLICATE KEY UPDATE`), visit lookup by `(day, vid)`, mark flags, add seconds, report queries, rollup query, batched delete |
| `AnalyticsTracker` | Rules for one track request: off / bot / team member checks, today's day, page normalizing, caps, writes through the repository. Also `mark_started(vid, pageUrl)`, `mark_asked(visit_id)`, `mark_talked(visit_id)` for the chat code |
| `AnalyticsReport` | Range report: combines `daily` rows (finished days) with live detail queries (today, untotalled days); median from buckets |
| `AnalyticsMaintenance` | Daily cron: rollup, then purge with the batch loop |
| `Rest\AnalyticsController` | `POST /analytics/track` (public), `GET /analytics/report`, `GET /analytics/pages`, `GET/POST /analytics/settings` (admin) |

### 5.2 `POST /analytics/track` (public)

```json
{ "vid": "Xq3…22", "device": "mobile",
  "events": [
    { "t": "view", "path": "/pricing/", "title": "Pricing" },
    { "t": "open", "path": "/pricing/", "title": "Pricing" },
    { "t": "time", "s": 37 }
  ] }
```
Response `{ "day": "2026-11-17" }` (or `{ "day": …, "off": true }` when analytics is off, so the browser stops).

Rules:
- No nonce (cached pages), like `/chat/ask`. `nocache_headers()`.
- **Skipped silently** (still answers `{day}`): analytics off; bot user agent (regex: `bot|crawl|spider|slurp|
  facebookexternalhit|headless|lighthouse|pingdom|…`, filter `mnaisp_analytics_is_bot`); team member
  (`wp_validate_auth_cookie` + `user_can($id, LiveChatService::capability())`) unless `include_team`.
- Rate limit: `RateLimiter::allow('track', 30)` per IP-hash per minute.
- Caps: ≤ 10 events per request; `time` 1–300 s per event, `active_seconds` ≤ 86,400 per visit; ≤ 100 page rows per visit
  per day; path ≤ 500 chars, title ≤ 200 (sanitized).
- `view` = upsert visit + upsert page. `open` = same + `pages.opened = 1` + set `opened_at` / `first_open_page` only when
  empty. `time` = add seconds to the visit (creates it if missing).
- Path: `wp_parse_url($url, PHP_URL_PATH)` of a same-site URL only, trailing slash kept as WordPress makes it,
  lowercased host check like `ConversationLogger::same_site_url()`.

### 5.3 Hooks into existing code

| Place | Change |
|---|---|
| `ChatController::ask` + `ask_human` + `ask_person` | Accept optional `visitorId` (same pattern). After honeypot passes: `tracker->mark_started(vid, pageUrl)` (upsert visit for today, `started = 1`; upsert the page row
from `pageUrl`, already sent today, and set `first_msg_page` only when empty), then `conversations.visit_id = visit.id` when the conversation exists |
| `LiveChatService::request_person` | After the event row: `tracker->mark_asked(conversation.visit_id)` |
| `LiveChatService::reply` | After the agent row: `tracker->mark_talked(conversation.visit_id)` |
| `ChatConfig::to_array` | `analytics: { enabled, track: true }`; `track: false` for team members when not included (page caches usually skip logged-in users; the server check is the real one) |
| `Plugin` | Wire services, register controller, cron event + schedule, privacy text paragraph |
| `Lifecycle` | New option + 3 tables in uninstall, clear the new cron hook |
| `Tables` / `Migrations` | 3 tables + `conversations.visit_id`, VERSION 5 |

Started counts even when recording is off (no conversation row): `mark_started` needs only the visitor ID. With
recording off there is no live chat, so `asked` / `talked` stay 0 (user's call: no note).

---

## 6. Browser

### 6.1 Loader (`loader.ts`, every widget page) — new small module `widget/analytics.ts`

Kept tiny, no React, no imports from chat modules (same rule as `storedChatIsLive()`).

- `track(event)`: puts the event in a queue; flushes after 1 s with `fetch(…, { keepalive: true })`, or with
  `navigator.sendBeacon` on `pagehide` / `visibilitychange: hidden`.
- **View:** after the page is idle (`requestIdleCallback`, fallback 2 s), if this page key is not in `sent` for today.
- **Open (floating):** in `setOpen(true)` when the visitor caused it: button click, launcher label, notification click
  (`mnaisp:open`). The closed auto-mount (`storedChatIsLive`) is not an open. Once per page per day (`opened` list).
- **Open (shortcode):** first `pointerdown` or `keydown` inside `.mnaisp-chat-inline` (capture listener, once).
- **Skip all** when `navigator.webdriver`, analytics off in config, or `track: false`.
- Device from `userAgentData.mobile` / UA (same checks as "Smart search on visitor devices").
- Exposes the vid to the chat bundle via `window.mnaispAnalytics = { vid(), time(seconds) }` (avoids the double-load
  problem of importing the loader from the chat bundle).

### 6.2 Chat bundle — time tracker (`widget/useActiveTime.ts`)

- Counts while: panel open (floating `open`, inline: in viewport) **and** `!document.hidden` **and** last activity ≤ 60 s ago.
- Activity: `keydown`, `pointerdown`, `pointermove`, `wheel`/`scroll` inside the chat; a new message arriving
  (answer stream start/end, team reply, event) — hooked in `useChat`.
- Ticks once per second in memory; sends the collected seconds via `mnaispAnalytics.time(s)` on panel close, tab hidden,
  `pagehide`, and every 2 min while counting (so a crash loses at most 2 min).
- Not in admin Preview (Preview doesn't use `useChat`; tracker mounts only from `mount.tsx`).

### 6.3 `useChat`

`/chat/ask` and `sendToTeam` bodies get `visitorId: window.mnaispAnalytics?.vid() ?? ''`.

### Requests per visitor (typical)

| Action | Requests |
|---|---|
| Visit 5 pages, never open chat | 5 (first visit of each page today); a repeat visit of a page today: 0 |
| Open chat, ask 3 questions, 4 min | + 1 open + 2–3 time flushes; questions add none (they ride on `/chat/ask`) |

---

## 7. Rollup and purge (cron)

New daily event `mnaisp_analytics_daily` (scheduled in `admin_init` like the conversation purge).

```
1. rollup:
   days = every day < today (site) that has detail rows and
          (no daily row yet OR day ≥ today − 2)          ← late time flushes / replies after midnight
   for each day: REPLACE INTO daily (site row + one row per page) FROM visits/pages     (idempotent)
2. purge (retention_days > 0):
   cutoff = today − retention_days; only days that have a daily row
   loop:
     DELETE FROM pages  WHERE day < cutoff LIMIT 5000   (own statement, autocommit)
     DELETE FROM visits WHERE day < cutoff LIMIT 5000
     both < 5000 → done
     run time > 20 s → wp_schedule_single_event(+120 s, mnaisp_analytics_daily), stop
```
Retention 0 = no purge (details kept forever), rollup still runs (reports stay fast).

Per-page `first_opens` in the rollup = visits whose `first_open_page` points at a pages row with that `page_key`.
Per-page `started` / `asked` / `talked` = visits whose `first_msg_page` points at that page (and have the flag).
Each started visitor counts on exactly one page.

---

## 8. Reports

### `GET /analytics/report?from=YYYY-MM-DD&to=YYYY-MM-DD[&page=<page_key>]`

```json
{ "range": { "from": "…", "to": "…" },
  "funnel": { "visitors": 1000, "opened": 180, "started": 120, "talked": 15, "asked": 22 },
  "time": { "total": 29700, "avg": 165, "median": 80,
            "ai": { "avg": 140, "median": 70 }, "person": { "avg": 420, "median": 360 } },
  "daily": [ { "day": "2026-11-11", "visitors": 140, "opened": 25, "started": 17, "talked": 2 }, … ] }
```
- Range = sum of days (user's call). Max range 366 days; default last 30 days.
- With `page`: the page funnel: visitors on the page → opened on the page → started (first message of the day sent on
  the page) → asked / talked (same visitors). Page funnels of started visitors add up to the site funnel.
- Days from `daily`; today and untotalled days from detail queries (`GROUP BY day` on `visits` / `pages` with the `day`
  indexes).

### `GET /analytics/pages?from&to&page=1&per_page=20`

Rows `{ page_key, path, title, first_opens, first_open_share, visitors, opened, open_rate }`, sorted by `first_opens`,
plus `total`. `first_open_share` = first opens ÷ all first opens in the range ("Share of first opens");
`open_rate` = opened on the page ÷ unique visitors on the page. Title = latest title seen for that path.

---

## 9. Admin tab "Analytics"

```
Analytics                                         [Today] [7 days] [30 days] [90 days] [Custom ▾]
┌ Funnel ───────────────────────────────────────────────────────────────────────────┐
│ Unique visitors        ████████████████████████████████████  1,000   100%         │
│ Opened chatbot         ███████                                  180    18%         │
│ Started conversation   ████                                     120    67% · 12%   │
│ Talked to a person     ▌                                         15    13% · 1.5%  │
│ Asked for a person: 22 (7 not answered)                                            │
└────────────────────────────────────────────────────────────────────────────────────┘
┌ Avg time ┐ ┌ Median time ┐ ┌ Total time ┐   AI only: avg 2 m 20 s · median ≈ 1 m 10 s
│ 2 m 45 s │ │ ≈ 1 m 20 s  │ │ 8 h 15 m   │   With a person: avg 7 m · median ≈ 6 m
└──────────┘ └─────────────┘ └────────────┘
┌ Daily trend (visitors / opened / started / talked) ────────────── small SVG chart ┐
┌ First-open pages ─────────────── page · first opens · share · visitors · open rate ┐
│ click a row → funnel for that page (drawer)                     [Show all]       │
(Settings live on the Settings tab: AnalyticsSettingsCard. Off → Analytics tab shows "Analytics is off" + link to Settings.)
```
- `pages/Analytics.tsx`, `api/analytics.ts`, admins only (not `live`).
- Charts: own small SVG components (funnel bars are plain divs), no chart library (user's call). Trend chart: 4 lines
  (visitors / opened / started / talked), day axis, hover tooltip with the day's numbers, legend toggles, light/dark colors
  from the admin theme tokens.
- Empty state: "No visitors counted yet. Numbers appear after visitors load a page with the chatbot."
- Today's numbers: loaded on open + Refresh button (no polling).

---

## 10. Privacy

- Privacy-policy paragraph (added in `privacy_policy_text()`): an anonymous ID kept in the browser for one day, which
  pages with the chat were visited, whether the chat was opened/used, time spent; no IP, no message text; detail kept
  N days (or until deleted).
- No personal data in analytics tables (no user ID, no IP), so no GDPR exporter/eraser needed.
- Paths only (no query strings, no domain).

---

## 11. Milestones

| # | What | Done when |
|---|---|---|
| **A0** | Schema v5 (3 tables + `conversations.visit_id`), `Tables`, `AnalyticsSettingsService`, `AnalyticsRepository` (writes), uninstall | Dev DB migrated (backup first), tables exist |
| **A1** | `AnalyticsTracker` + `POST /analytics/track` + hooks in ask / request_person / reply + `ChatConfig.analytics` | WP-CLI checks: view/open/time upserts, bot/team skipped, caps, started/asked/talked marked |
| **A2** | Browser: `widget/analytics.ts` in loader (view, open floating + shortcode, queue, beacon, day/vid rotation), `useActiveTime`, `visitorId` in `/chat/ask` | Browser check (Playwright throwaway script, like L2/L3): rows appear, repeat page view sends nothing, idle stops the clock |
| **A3** | `AnalyticsMaintenance`: rollup + batched purge + cron | WP-CLI: fake old rows rolled up, purge with budget + follow-up event, retention 0 keeps details |
| **A4** | `AnalyticsReport` + report/pages/settings REST + Analytics tab (range, funnel, time, trend, pages, page funnel drawer) + `AnalyticsSettingsCard` on Settings | Numbers match a hand count on dev data; funnel steps never grow |
| **A5** | Privacy text, docs (`docs.html` schema, this plan's deviations), WWC note | Docs updated, commit |

Tests: none new (user's call: tests later). Checks by WP-CLI scripts + throwaway Playwright scripts, not committed.

---

## 12. Review questions (all answered 2026-11-17)

1. ~~**Per-page funnel**~~ ANSWERED 2026-11-17 (alternative): visitors on that page → opened on that page → started =
   first message of the day sent on that page → asked / talked credited to that same page. Needs only
   `visits.first_msg_page` + the `pageUrl` `/chat/ask` already sends (no extra rows).
2. ~~**Charts**~~ ANSWERED 2026-11-17: own small SVG components, no chart library.
3. ~~**Median**~~ ANSWERED 2026-11-17: approximate median from the time buckets, shown with "≈", for every range.
4. ~~**Time flush**~~ ANSWERED 2026-11-17: send every 2 min while counting, plus on close / tab hidden / pagehide.
5. ~~**Store device**~~ ANSWERED 2026-11-17: store now (`visits.device`), and the site daily row keeps per-device counts
   (`devices` JSON), so the P1 device split also works after detail rows are deleted. No device UI yet.
6. ~~**On by default**~~ ANSWERED 2026-11-17: on for new installs and for existing sites after the update.
7. ~~**Settings place**~~ ANSWERED 2026-11-17: Settings tab, own card `AnalyticsSettingsCard` (after `WidgetSettingsCard`, own Save
   like the other cards). Analytics tab links to it when analytics is off.
8. ~~**Endpoint cost**~~ ANSWERED 2026-11-17: normal REST endpoint; lighter endpoint only later if a big site needs it.

All review questions answered 2026-11-17. Next = A0.

## 13. Built (2026-11-17): deviations from the plan

Commits: A0 5632580, A1 0d55a7b, A2 f4c3ad1, A3 8d96a82, A4 b488340, A5 (privacy text).

- **Visitor ID rotation:** The browser computes the site day itself from `analytics.tzOffset` (site time zone offset, in the widget config) and makes a new ID before sending on a new day. The server's `{day}` only corrects the stored day (same ID), so one day never mixes two IDs.
- **Track request body:** Form-encoded (`events[0][t]=…`), not JSON: `sendBeacon` can't send a JSON content type reliably. Same body for `fetch keepalive`.
- **Track response `off`:** Also for bots and excluded team members (not only analytics off): the browser stops sending from that page.
- **Asked vs talked:** The funnel shows "Asked for a person: N · talked to a person: M" without a "not answered" number: a team member can also reply in chats where nobody asked, so asked − talked would be wrong.
- **Time counts for openers only:** Seconds of visitors without an open are left out of totals and averages.
- **Shortcode time:** Counted only after the visitor's first interaction with the inline chat (being on screen alone is not using the chat).
- **Open records the view:** An open also marks the page as reported, so no separate view is sent for it.
- **Columns:** `time_buckets` and `devices` are TEXT (not VARCHAR(255)) so big sites never cut the JSON. Device '' is stored as `unknown` in `devices`.
- **Tab place:** Analytics tab sits after Live chat.
- **Not tested:** The purge time budget + follow-up event path (needs a backlog that takes over 20 s); batches and the loop were tested with 24,000 rows.
