# Lead Form in Chat — Implementation Plan

> **Goal:** build the lead form described in `docs/lead-form-requirements.md`: an owner-built form (Text, Number,
> Email, Textarea fields) shown inside the chat after 2 answers, sooner when the bot can't answer or nobody is online
> for a person request; closable (never shown by itself again), reopenable (collapsed line, header icon, link under
> a later "I don't know"); sent by a team member during live chat; leads stored until the admin deletes them, emailed,
> listed in a Leads tab with CSV export.
>
> **Builds on:** widget inline config (`ChatConfig` → `window.mnaispChat.config.widget`), `useChat` entries in
> `sessionStorage` (`mnaisp-chat-v1`), live polling `POST /chat/live` (`LiveChatService::visitor_poll`, event rows in
> `wp_mnaisp_messages`), `HandoffDetector` + `LiveChatService::request_person` (`REQUEST_*` states), `AlertService` /
> `AlertDispatcher` / `EmailChannel`, `RateLimiter::allow()` buckets, the widget honeypot (`website`), `Migrations`
> (schema v5), React admin tabs (`App.tsx`), Style tab preview (`widget/Preview.tsx`).
>
> **Not in this plan:** CRM / webhook integrations (D4), more field types, conditional fields, buying-word trigger,
> lead analytics (shown / submitted / closed counts).

Created: 2026-11-18 · Status: **built 2026-11-18 (LF0–LF6), see section 13** (review questions answered, section 12)

---

## 1. Summary

| Item | Decision (proposed) |
|---|---|
| Form definition | One option `mnaisp_lead_form`: switch, trigger, texts, consent, list of fields with stable IDs |
| Field types | `text`, `number`, `email`, `textarea` (requirements 4.1). Max 10 fields, at least 1 |
| Where triggers run | In the **browser** (`useChat`): it already knows answers, "I don't know", handoff state, live mode. No extra request |
| "Never again by itself" | `localStorage` key `mnaisp-lead-v1` = `{ done: 'closed' \| 'submitted', at }` (lasts across tabs and visits) |
| "Not twice in one chat" | A lead card entry already in this conversation's entries (`sessionStorage`) |
| Card in the chat | New chat entry `role: 'lead'`, stored with the other entries, so it survives page loads in the tab |
| Storage | New table `wp_mnaisp_leads` (schema v6): answers as JSON with label snapshots + search columns |
| Link to the chat | By **conversation UUID** (works before the first question and with recording off; joined to `conversations` when it exists) |
| Live chat | Agent button → event row `lead_requested` → visitor poll → card opens. Visitor result → events `lead_submitted` / `lead_closed` for the agent |
| Email | New alert type `lead`, setting "Email me new leads" (default on), same recipients as other alerts |
| Admin | New tab **Leads** (admins only): list + detail + delete + CSV, and a **Form** sub-tab (builder, trigger, texts, preview) |
| Privacy | WP personal-data exporter + eraser for leads (by email), privacy-policy paragraph, no IP |
| Free / Pro | None, one plugin (requirements Q7) |

---

## 2. Flow

```
Visitor's browser (useChat)                         Server                               Team member (Live chat)
─────────────────────────────                       ──────                               ───────────────────────
answer finished ──► shouldOffer()?
  - feature on, not done (localStorage)
  - no lead entry in this chat
  - mode = ai (no team member in charge)
  - answers ≥ N  OR dont_know  OR handoff offline/off
  - not streaming
        │ yes
        ▼
push entry {role:'lead', state:'open', trigger}
LeadCard renders fields from config
        │
   ┌────┴─────────────┐
 Send                 ✕ / No thanks
   │                    │
POST /chat/lead ──────► validate vs current fields     POST /chat/lead/closed ─► event lead_closed
   │                    rate limit + honeypot             (only if conversation exists)
   │                    [event lead_submitted] ──────────────────────────────► thread shows answers
   │                    insert lead (event_id)
   │                    queue alert (email)
   ▼                    ◄── { ok, leadId }
entry state 'sent' → thank-you line                                              "Ask for contact details"
localStorage done = submitted                                                    POST /live/chats/{id}/lead-form
                                                                                        │
poll /chat/live ◄─────── event lead_requested ◄──── add_event(lead_requested, agent) ◄──┘
push entry {role:'lead', trigger:'agent'}  (opens even when done)
```

---

## 3. Form settings (`mnaisp_lead_form` option, autoloaded)

New `includes/Leads/LeadFormSettingsService.php` (same pattern as `WidgetSettingsService`: `defaults()`, `get()`,
`save()` with `ValidationException`).

```php
array(
  'enabled'           => true,                 // Whole feature: triggers, header icon, agent button.
  'trigger'           => 'after_answers',      // after_answers | dont_know | pre_chat | manual
  'after_answers'     => 2,                    // 1..10
  'pre_chat_required' => false,                // pre_chat only: true = no ✕, input locked until sent
  'intro'             => 'Want us to follow up? Leave your details.',
  'submit_text'       => 'Send',
  'thanks_text'       => "Thanks! We'll be in touch.",
  'consent'           => array( 'enabled' => false, 'text' => 'I agree to the privacy policy.' ),
  'fields'            => array(
    array( 'id' => 'f_name',  'type' => 'text',  'label' => 'Name',  'placeholder' => '', 'required' => true ),
    array( 'id' => 'f_email', 'type' => 'email', 'label' => 'Email', 'placeholder' => '', 'required' => true ),
    array( 'id' => 'f_phone', 'type' => 'text',  'label' => 'Phone', 'placeholder' => '', 'required' => false ),
  ),
)
```

| Trigger value | Shows the form by itself | Early triggers (dont_know, handoff offline) |
|---|---|---|
| `after_answers` (default) | After N finished answers in this chat | Yes |
| `dont_know` | Never on count | Yes |
| `pre_chat` | Before the first message (instead of on count) | Yes, if skipped and not done |
| `manual` | Never | No: only header icon and team member |

`enabled = false` replaces the requirements' "Off" option (also hides the header icon and the agent button).
Two settings, not one select (user's call 2026-11-18): switch "Show the lead form" + select "When to show it".

Field rules (`save()`):

- `id`: `f_` + 8 random chars, made by the builder; kept when the label changes. Server rejects duplicates / bad format.
- `type` ∈ `text | number | email | textarea`; `label` 1..100 chars; `placeholder` ≤ 100; `required` bool.
- `number` only: optional `min` / `max` (numbers, `min ≤ max`).
- 1..10 fields. Texts sanitized with `sanitize_text_field`; consent text allows a `[text](https://…)` link like the
  footer note (`WidgetSettingsService::LINK_PATTERN`), and `{privacy}` = the site's privacy page link
  (`get_privacy_policy_url()`).

Sent to the widget in `ChatConfig` as `widget.leadForm` (camelCase, no server-only data). Missing in cached pages
before the feature: the widget treats it as off.

---

## 4. Database (schema v6)

### 4.1 `wp_mnaisp_leads` — one row per submitted form

```sql
CREATE TABLE wp_mnaisp_leads (
  id bigint(20) unsigned NOT NULL AUTO_INCREMENT,
  conversation_uuid char(36) DEFAULT NULL,      -- the widget's conversation ID (may have no conversations row)
  event_id bigint(20) unsigned DEFAULT NULL,    -- messages row "lead_submitted" (when the conversation exists)
  answers longtext NOT NULL,                    -- JSON [{id, type, label, value}] in form order, label as sent
  email varchar(191) DEFAULT NULL,              -- first Email answer, lowercased (search, export, privacy eraser)
  name varchar(191) DEFAULT NULL,               -- first Text answer (list column, email subject)
  consent tinyint(1) NOT NULL DEFAULT 0,
  trigger_reason varchar(20) NOT NULL,          -- after_answers | dont_know | handoff | pre_chat | visitor | agent
  page_url varchar(500) DEFAULT NULL,           -- without query string, like analytics pages
  user_id bigint(20) unsigned DEFAULT NULL,     -- logged-in visitor (same cookie check as ConversationLogger)
  created_at datetime NOT NULL,                 -- UTC
  PRIMARY KEY  (id),
  KEY created_at (created_at),
  KEY email (email),
  KEY conversation_uuid (conversation_uuid)
)
```

- Kept until the admin deletes them: **no cron purge** (requirements Q6). Conversation purge does not touch leads;
  a lead whose conversation was purged just loses its transcript link.
- `Tables::LEADS`, added to `Tables::ALL` (uninstall drops it), `mnaisp_lead_form` added to the uninstall options.
- `trigger_reason` = what showed the card that was sent (`visitor` = header icon / "Fill in" / link).

### 4.2 `wp_mnaisp_messages` — three new event names (no schema change)

| Event (`reason`) | Written when | `user_id` | Visitor poll |
|---|---|---|---|
| `lead_requested` | Team member pressed "Ask for contact details" | agent | **Sent** (opens the card) |
| `lead_submitted` | Visitor sent the form (conversation exists) | null | Not sent |
| `lead_closed` | Visitor closed the form (conversation exists) | null | Not sent |

Event rows don't count as messages (existing rule). `visitor_poll` filters out `lead_submitted` / `lead_closed`.

---

## 5. Server

### 5.1 New classes (`includes/Leads/`)

| Class | Job |
|---|---|
| `LeadFormSettingsService` | Option read / validate / save (section 3), `public_config()` for the widget |
| `LeadRepository` | insert, find, list (search, date range, paging), delete many, by conversation UUID, by email (privacy) |
| `LeadService` | `submit()` (validate answers vs current fields, honeypot, rate limit, event row, insert, alert), `closed()` |
| `LeadPrivacy` | `wp_privacy_personal_data_exporters` / `_erasers` callbacks (by email), privacy-policy paragraph |
| `LeadCsv` | CSV rows: fixed columns + one column per label seen in the range (union, form order of the current form first) |

### 5.2 Public REST (widget)

**`POST /chat/lead`** (`permission_callback: __return_true`, like `/chat/ask`)

```json
{ "conversationId": "uuid", "pageUrl": "https://…", "trigger": "after_answers",
  "answers": { "f_name": "Sam", "f_email": "sam@example.com" }, "consent": true, "website": "" }
```

1. Feature on, else `403`. Honeypot `website` filled → `200 { ok: true }` and nothing saved (bots learn nothing).
2. `RateLimiter::allow('lead', 3)` per minute and a per-visitor daily cap (filter `mnaisp_leads_per_day`, default 10) → `429`.
3. Validate against the **current** fields: unknown IDs dropped; required empty → field error; `email` → `is_email`;
   `number` → numeric + min/max; `text` ≤ 200, `textarea` ≤ 2,000; consent required when enabled.
   Errors → `422 { fields: { f_email: "Enter a valid email." } }` (widget shows them under the fields; a cached page
   with an old form gets "This form changed, please reload" when a required current field is missing).
4. Conversation row by UUID? → `add_event(lead_submitted)` first, keep its ID.
5. Insert lead (`answers` JSON with label snapshots, `email`, `name`, `event_id`, `user_id`, page without query).
6. `AlertService::lead()` (section 5.5). → `200 { ok: true, id }`.

**`POST /chat/lead/closed`** `{ conversationId, trigger }` → `add_event(lead_closed)` when the conversation exists and
the last lead event in it is not already `lead_closed`; else nothing. Rate limited (`lead_closed`, 5/min). Always `204`.

### 5.3 Admin REST (`can_manage`, admins)

| Route | Job |
|---|---|
| `GET /lead-form` · `POST /lead-form` | Form settings (builder) |
| `GET /leads?search=&from=&to=&page=&per_page=` | List: id, created, name, email, page, trigger, conversation link (id when the row exists) |
| `GET /leads/{id}` | All answers + conversation summary (id, message count) |
| `DELETE /leads` `{ ids: [] }` | Delete one or many (max 500 per call) |
| `GET /leads/export?search=&from=&to=` | `text/csv` download (UTF-8 BOM for Excel), same filters as the list |

### 5.4 Live chat (`can_live_chat`, admins + editors)

**`POST /live/chats/{id}/lead-form`** → `LiveChatService::ask_lead()`: chat must be in `human` mode with this agent
in charge (same check as `reply`), feature on → `add_event(lead_requested, agent)`. Returns the thread row so the
dashboard shows "Contact form sent" at once.

`thread()` / `MessagePresenter`: `lead_submitted` rows carry the lead's answers (`LeadRepository::by_event_ids()`,
one query per thread page); `lead_closed` → "Visitor closed the contact form".

### 5.5 Hooks into existing code

| Where | Change |
|---|---|
| `ChatConfig::widget` | `leadForm` from `LeadFormSettingsService::public_config()` |
| `ChatController` streamed answer | Final payload gets `handoff` (`REQUEST_*`) also when the **AI marker** triggered `request_person` (today only the phrase path returns it); `AnswerResult` carries it |
| `LiveChatService::visitor_poll` | Skip `lead_submitted` / `lead_closed` rows |
| `AlertService` | `lead( int $lead_id )`: `Alert::TYPE_LEAD`, title "New lead: {name or email}", answers in `details`, recent transcript when the conversation exists, link to the Leads tab (`#/leads/{id}`). Setting `alert_on_lead` (`always` / `never`, default `always`) in `ConversationSettingsService`. Cap: filter `mnaisp_alerts_leads_per_hour` (default 30). Not tied to "team online" |
| `ConversationsController` single conversation | `leads` (by UUID) for the viewer marker |
| `Plugin` | Register controllers, privacy exporter / eraser, policy paragraph in `privacy_policy_text()` |
| `Migrations` | `VERSION = 6`, leads table in `schema()` |

---

## 6. Widget (chat bundle; the loader stays unchanged)

### 6.1 Types and storage

- `ChatEntry.role` adds `'lead'`, with `lead: { state: 'open' | 'skipped' | 'sent', trigger }`.
  `completedTurns()` / `contextQuery()` skip `lead` entries (like `system`), so the AI never sees them.
- `widget/lead.ts`: `leadDone()` / `markLeadDone('closed' | 'submitted')` on `localStorage` `mnaisp-lead-v1`
  (try/catch: blocked storage = treat as not done, the per-chat rule still stops repeats).
- Typed answers are kept in the entry while the card is open (page change in the same tab keeps them).

### 6.2 When the card is added (`useChat`)

`offerLead(trigger)` adds `{ role: 'lead', state: 'open' }` at the end of the entries when **all** hold:
feature on · `live.mode !== 'human'` · not `leadDone()` · no `lead` entry in this chat · not busy (answer finished).

| Trigger (setting) | Checked after |
|---|---|
| `after_answers` | An answer finishes `done` / `dont_know` and the count of finished AI answers in this chat ≥ N |
| `dont_know` (early) | An answer finishes `dont_know` |
| `handoff` (early) | `handoff` = `offline` or `off` in the answer payload, or poll event `request_expired` |
| `pre_chat` | Chat opens with no entries: card shown first (before welcome). `pre_chat_required` → no ✕, input disabled until sent |

Always (when the feature is on, any mode incl. live chat), not limited by "done" or "once per chat":

- **Header icon** (contact card, before the "new conversation" button): open card in this chat → scroll to it and
  focus the first field; else add a new card (`trigger: 'visitor'`).
- **"Fill in"** on a skipped card → back to `open`.
- **"Leave your details"** link under a `dont_know` answer when `leadDone() === 'closed'` (only after closing, not
  after submit) → same as the header icon.
- **Poll event `lead_requested`** → add a new open card (`trigger: 'agent'`), even when done; unread count +1 when
  the chat is closed.

### 6.3 `LeadCard` (`widget/LeadCard.tsx`)

- Fields from `config.widget.leadForm.fields`: `<input type="text|number|email">` / `<textarea>`, labels tied to
  inputs, `required`, `min`/`max`, `autocomplete` hints (`name`, `email`, `tel` when the label looks like phone).
- Browser checks first (same rules as the server), server field errors shown under each field.
- Send → `POST /chat/lead` → state `sent`, thank-you line, `markLeadDone('submitted')`.
- ✕ / "No thanks" → state `skipped` (one line "Contact form skipped · Fill in"), `markLeadDone('closed')`,
  `POST /chat/lead/closed` (only once the conversation exists; fire-and-forget).
- Focus moves to the first field when the card is added by the visitor / agent (not for automatic triggers, so
  typing in the chat input is never interrupted). Esc on the card = close.
- Styles in `widget.css` (`mnaisp-lead-*`), using the widget colour variables: bot bubble background, accent border,
  send button colours.
- "New conversation" clears entries (card included) but not `mnaisp-lead-v1`.

### 6.4 Requests per visitor (typical)

0 extra while chatting (triggers run in the browser). 1 on submit, ≤ 1 on close. Lead events ride the existing poll.

---

## 7. Admin

### 7.1 Tab "Leads" (`#/leads`, admins only)

Sub-tabs **Leads** and **Form** (same pattern as other tabs with sections).

**Leads**: table (date, name, email, page, how shown, conversation link), search (name / email / answers text),
date range presets like Analytics, select rows → Delete (confirm dialog), Export CSV (current filters).
Row click → side sheet (`ui/sheet.tsx`) with every answer (label as sent), consent, page, link to the conversation
(Conversations tab) when it exists. Empty state explains the trigger setting.

**Form**: left = builder, right = live preview (the real `LeadCard` inside the widget Shadow DOM, like the Style
preview).

- Switch "Show the lead form", trigger select (+ N for after answers, + "Required before chatting" for pre-chat).
- Texts: intro, send button, thank-you. Consent switch + text.
- Field list: add (type menu: Text, Number, Email, Textarea), edit (label, placeholder, required, min/max for
  Number), reorder (up / down buttons, keyboard friendly; drag later if wanted), delete (confirm when leads have
  answers for it: "Old leads keep their answers").
- Save with field errors from the server, like the Style tab.

### 7.2 Live chat screen

- Button **"Ask for contact details"** next to the reply box, only while this agent is in charge and the feature is on.
  Disabled for 10 s after pressing (no double send).
- Thread rows: "Contact form sent" · "Visitor left contact details" (answers inline) · "Visitor closed the contact form".

### 7.3 Conversations viewer

Marker row at the lead's place in the transcript (event row) with the answers; for leads without an event row
(sent before the first question), a box above the transcript "Left contact details".

### 7.4 Settings → Alerts card

New select "Email me new leads": Always / Never (default Always).

---

## 8. Privacy

- Exporter "Chatbot leads": all leads with that email (`email` column) → every answer, page, date.
- Eraser: deletes those leads (not anonymize: a lead is only personal data).
- Policy paragraph (added to the existing `privacy_policy_text()`): what the form collects, why, kept until deleted,
  who gets the email.
- No IP stored. Page URL without query string.

---

## 9. Edge cases

| Case | Behaviour |
|---|---|
| Recording off | Lead saved and emailed; no conversation row → no event, no transcript, no link (requirements Q4) |
| Form submitted before the first question (pre-chat, header icon) | Lead has `conversation_uuid`; the link appears once the conversation is recorded |
| Owner changes fields while a visitor has the card open | Server validates against the current form; missing new required field → "This form changed, please reload" |
| Owner deletes a field | Old leads keep `{label, value}` snapshots; CSV shows the old label as its own column |
| Agent sends the form twice | Second card only if no open card in the visitor's chat (else focus the open one) |
| Live chat hand-back | Card stays where it is; triggers apply again (mode `ai`) |
| `localStorage` blocked | Not remembered across visits; still once per chat |
| Shortcode chat | Same card and triggers; header icon in the inline header |
| Bots | Honeypot + rate limits; silent success on honeypot |

---

## 10. Milestones

| # | Scope | Done when |
|---|---|---|
| **LF0** | Schema v6 leads table, `Tables::LEADS`, uninstall list, `LeadFormSettingsService` (+ tests later), `LeadRepository` | Table created on upgrade, settings round-trip |
| **LF1** | `POST /chat/lead`, `/chat/lead/closed`, `ChatConfig.leadForm`, handoff state in streamed answers, poll filter, `AlertService::lead`, alert setting, privacy exporter / eraser / policy text | curl submit saves a lead and sends the email |
| **LF2** | Widget: `lead` entry, `LeadCard`, triggers, `mnaisp-lead-v1`, header icon, "Fill in", "Leave your details" link, pre-chat | Full visitor flow on the dev site, floating + shortcode |
| **LF3** | Admin Form sub-tab: builder, trigger, texts, consent, live preview | Owner builds a form, the widget uses it |
| **LF4** | Admin Leads sub-tab: list, search, range, detail sheet, delete, CSV; Conversations viewer marker | Leads listed, exported, deleted |
| **LF5** | Live chat: agent button, `lead_requested` → card, thread rows for submitted / closed | Agent asks, visitor fills, agent sees answers |
| **LF6** | Docs: docs.html (leads table, events), CLAUDE.md, build + lint + PHPCS | Clean build, docs updated |

Master only; WWC branch merge only when asked.

---

## 11. Files

New: `includes/Leads/{LeadFormSettingsService,LeadRepository,LeadService,LeadPrivacy,LeadCsv}.php`,
`includes/Rest/{LeadController,LeadFormController}.php`, `assets/src/widget/{lead.ts,LeadCard.tsx}`,
`assets/src/api/leads.ts`, `assets/src/pages/Leads.tsx`, `assets/src/components/leads/{LeadsList,FormBuilder,LeadSheet}.tsx`.

Changed: `Migrations.php`, `Tables.php`, `uninstall.php`, `Plugin.php`, `ChatConfig.php`, `ChatController.php`,
`AnswerResult.php`, `LiveChatService.php`, `LiveChatController.php`, `MessagePresenter.php`,
`ConversationsController.php`, `AlertService.php`, `Alert.php`, `ConversationSettingsService.php`,
`widget/{types.ts,useChat.ts,ChatPanel.tsx,live.ts,widget.css}`, `App.tsx`, `pages/LiveChat.tsx`,
`components/ConversationViewer.tsx`, alert settings card.

---

## 12. Review questions

1. ~~**Off vs Manual**~~ ANSWERED 2026-11-18: keep it **split**: switch "Show the lead form" (off = no form, no
   icon, no agent button) + "When to show it" select with `manual` (only header icon + team member).
2. ~~**Leads tab for admins only**~~ ANSWERED: yes. Editors send the form and see answers in the live thread only.
3. ~~**Pre-chat required**~~ ANSWERED: yes (no ✕, input locked), as an option; default not required.
4. ~~**Email**~~ ANSWERED: yes, default Always, same addresses.
5. ~~**Spam limits**~~ ANSWERED: yes, 3/min and 10/day per visitor.
6. ~~**Header icon**~~ ANSWERED: yes, from the start.
7. ~~**Consent checkbox**~~ ANSWERED: yes, off by default, with an EU / GDPR hint next to the setting.

(Questions 2–7 answered 2026-11-18 by "ok, start building" = the recommendations.)

---

## 13. Built (2026-11-18): deviations from the plan

Commits on master: LF0 8a08a1d + 9fe0c18, LF1 698bddf, LF2 7969994, LF3 1b28e1e, LF4 fac6184, LF5 2026fd3.

- **Validation errors** use the plugin's existing `400` with `fields` (not `422`).
- **CSV export** returns JSON `{ filename, csv }`; the admin app saves it as a file (REST would JSON-encode raw CSV).
  Cells starting with `= + - @` get a leading `'` (spreadsheet formula injection); phone `+1 555` exports as `'+1 555`.
- **Email setting** sits with the other email alerts (Live chat tab → Email alerts), with the same three choices
  (Never / Only when no one is online / Always), default Always.
- **Pre-chat card** shows after the welcome message, not before it. Suggested questions hide while a required
  pre-chat form is open.
- **Field delete** has no confirm dialog; the Fields card says old leads keep their answers.
- **Default field IDs** are fixed (`f_name0001`, `f_email001`, `f_phone001`); new fields get random IDs.
- **`RateLimiter::allow_daily()`** added for the per-visitor daily cap. `leads.event_id` got an index.
- **`lead_closed`** is written once in a row (closing again without a new request adds nothing).
- **`GET /live/chats`** returns `leadForm` (on/off) so the button shows only when the form is on.
- **Live thread and Conversations viewer** get the lead on the `lead_submitted` row (`lead`); the viewer also gets
  `leads` (sent before the first question, no event row) and shows them above the transcript.
- Not done: the existing `LifecycleTest` table list was already out of date (tests postponed by the user).
