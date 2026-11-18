# Lead Form in Chat — Requirements

Status: requirements only (2026-11-18). Architecture and implementation plan come later.
Visual version: `docs/lead-form-requirements.html`.
Related: commercial-requirements D2 (in-chat contact form) and D4 (lead export).

## 1. Goal

At some point in the chat, or near its end, the chatbot shows the visitor a small form **inside the chat**:
name, email and phone, with a short line asking them to fill it in. The site owner gets the visitor's contact
details together with what they asked about.

What the user asked for (2026-11-18):

1. Form shown inside the chat, with fields **name, email, phone**, and a text asking the visitor to fill it in.
2. Visitor can **close** the form without filling it in.
3. Once closed, it **never shows up again by itself**.
4. A visitor who closed it and keeps chatting must have a way to **open it again** later.
5. **When** to show the form is still undecided (main open question, see section 3 and 9).
6. (Added 2026-11-18) The form is **not hardcoded**: the site owner **builds** it from 4 field types
   (Text, Number, Email, Textarea). Name, email and phone are only the default form. See section 4.1.

## 2. Definitions

| Term | Meaning |
|------|---------|
| **Lead** | One submitted form: the visitor's answers to the owner's fields, consent, time, page URL, linked conversation. |
| **Field** | One item in the form builder: a type (Text, Number, Email, Textarea) and its settings (section 4.1). |
| **Shown** | The form card appeared in the chat (by a trigger or because the visitor opened it). |
| **Closed** | The visitor pressed the close (✕) / "No thanks" on the form. The form never shows by itself again for this visitor. |
| **Submitted** | The visitor sent the form and the server saved it. The form never shows by itself again. |
| **Trigger** | A rule that decides the form shows now (see section 3). |
| **Done** | Closed or submitted. A done visitor gets no automatic form; the reopen options in section 5 stay available. |

## 3. When to show the form (decided 2026-11-18)

Not at a fixed point. "End of chat" cannot be detected: visitors rarely say goodbye, they just leave. Asking
on the first message interrupts them before they got any value. So the form shows when it is **earned** or when
the moment **fits**, and the site owner picks the rule.

### 3.1 Default trigger

**After the visitor's 2nd question has been answered** (number set by the owner). By then the visitor got real
value, so asking for contact details feels fair.

Decided 2026-11-18: after 2 answers + can't answer + offline handoff. **No buying-word trigger for now**.

### 3.2 Early triggers (show sooner, whatever the count)

| Moment | Why | Exists in the plugin |
|--------|-----|----------------------|
| The bot **could not answer** ("I don't know") | Best moment: "I couldn't find that. Leave your details and the team will get back to you." | `dont_know` answer state |
| Visitor **asked for a person** and nobody is online | The team cannot reply now; contact details let them reply later | `HandoffDetector`, `requested_offline` |

### 3.3 Never show the form

- While an answer is still streaming (it appears after the answer finishes).
- **Automatically** while a team member is in the chat (live chat taken over). The team member decides instead
  (section 3.5). (Offline handoff = the visitor asked for a person but nobody is online; no live chat is running,
  so that trigger stays.)

### 3.5 Live chat: team member sends the form (decided 2026-11-18)

- During live chat no trigger shows the form by itself.
- The team member has a **"Ask for contact details"** button in the Live chat screen. Pressing it sends an event
  to the visitor's chat, and the form card opens there (same card, same fields).
- Works even if the visitor closed or submitted the form earlier: the team member asked on purpose.
- Delivered through the existing live chat polling, like a team reply. Saved as an event row in the conversation.
- The team member sees what happened in the conversation: "Contact form sent", then "Visitor left contact details"
  (with the answers) or "Visitor closed the form".
- The visitor can still close it; closing does not end or affect the live chat.
- The header contact icon stays available during live chat (the visitor's own choice).
- When the visitor is done (closed or submitted).
- Twice in the same chat.

### 3.4 Owner setting "Show the form"

| Option | Behaviour |
|--------|-----------|
| Off | No lead form. (Plan: a separate switch "Show the lead form", plus a "Manual" option here: header icon and team member only. User's call 2026-11-18.) |
| After N answers (default, N = 2) | Plus the early triggers of 3.2. |
| Only when the bot can't answer | "I don't know" and offline handoff only. |
| Before the chat starts | Pre-chat gate: the form shows first. Fields can be required to start chatting, or skippable. |

## 4. How the form looks and behaves

- Shown **inline in the transcript** as a bot message card, not a pop-up. The chat input stays usable: the visitor
  can ignore the form and keep typing.
- Card: short text (owner editable, e.g. "Want us to follow up? Leave your details."), the fields, a Send button,
  a ✕ / "No thanks" to close.
- Fields: built by the owner (section 4.1). Default: name, email, phone.
- Consent checkbox with a link to the privacy policy (GDPR). Owner can turn it off where not needed.
- Checks in the browser and again on the server: required fields filled, Email fields valid, lengths within limits.
- After submit: the card is replaced by a thank-you line ("Thanks, Sam! We'll be in touch."). Owner editable.
- Bot answers and team replies keep arriving below the card while it is open.
- Style follows the widget colours (Style tab).

### 4.1 Form builder (decided 2026-11-18)

The owner builds the form in admin: add, remove, reorder (drag or up/down), and edit fields. Four field types:

| Type | What the visitor sees | Settings | Server check |
|------|----------------------|----------|--------------|
| **Text** | One-line text box, `<input type="text">` (name, phone, company…) | Label, placeholder, required | Max 200 chars |
| **Number** | Number box, `<input type="number">` (team size, budget, quantity…); number keyboard on mobile | Label, placeholder, required, optional min / max | A number, within min / max |
| **Email** | Email box, `<input type="email">`; email keyboard on mobile | Label, placeholder, required | Valid email (`is_email`) |
| **Textarea** | Multi-line text box, `<textarea>` (message, "How can we help?") | Label, placeholder, required | Max 2,000 chars |

The text above the fields ("Want us to follow up? Leave your details.") is a **form setting** (intro text), not a
field type. Same for the Send button text and the thank-you text.

- Every field gets a **stable ID** when created. Renaming a label keeps old leads' answers attached.
- Each lead also stores the **label as it was** when sent, so deleting or renaming a field never breaks old leads.
- At least one field needed to save the form.
- Max 10 fields (keeps the card small in the chat).
- Preview of the form card in the builder, like the Style tab preview.
- Phone has no own type: it is a **Text** field, not Number (phone numbers have `+`, spaces, leading zeros;
  formats differ per country, so no format check).
- Consent checkbox stays a **built-in switch** (with privacy-policy link), not a field type.

Default form (new installs, owner can change everything):

| # | Type | Label | Required |
|---|------|-------|----------|
| 1 | Text | Name | yes |
| 2 | Email | Email | yes |
| 3 | Text | Phone | no |

Intro text: "Want us to follow up? Leave your details." 

## 5. Closing and reopening

### 5.1 On close

- The card shrinks to **one line** in the transcript: "Contact form skipped · Fill in". That line itself reopens
  the form, right where the visitor left it.
- No automatic form again for this visitor (all chats, all pages, later visits: stored in `localStorage`, not
  `sessionStorage`, so it lasts after the tab closes).

### 5.2 Ways to open it again (proposal)

1. **The "Fill in" link** on the collapsed line (above).
2. **A contact icon in the chat header**, always there (when the lead form is on), even after closing.
3. **Soft second chance**: when the bot later cannot answer, a small "Leave your details" link appears under that
   answer. Only a link, not the full form pushed again.

### 5.3 After submit

- No automatic form again for this visitor.
- Header icon still opens the form, so the visitor can correct or update details (stored as a new lead linked to
  the same conversation, newest shown first).

## 6. What the site owner gets

- **Stored lead**: new leads table, linked to the conversation (owner sees what the visitor asked) and the page
  where the form was sent.
- **Leads tab** in admin: list (date, page, link to the conversation, plus the answers; the first Email field and
  the first Text field as columns, the rest in the detail view), search, CSV export (one column per field label), delete (one or many).
- **Form builder** in admin (section 4.1).
- **Email for each new lead** through the existing alert system (`AlertService` / `EmailChannel`): details plus
  a short transcript and a link to the conversation.
- **Recording off** (conversation log switched off): leads are still saved and emailed, just without the
  conversation link and transcript (decided 2026-11-18). The form works the same for the visitor.
- **Conversation view** shows a lead marker ("Left contact details") with the details.
- **Analytics** (later, P1): forms shown, submitted, closed, so the owner can tune the trigger.

## 7. Privacy

- Lead data is personal data: added to the WordPress personal-data exporter and eraser (commercial-requirements I1).
- Privacy-policy text via `wp_add_privacy_policy_content`, like the conversation log and analytics.
- No IP address stored (same rule as the conversation log).
- Leads are **kept until the admin deletes them** (decided 2026-11-18). No retention setting, no automatic purge.
  The WordPress personal-data eraser still deletes a visitor's leads on request.

## 8. Quality requirements

- Form code loads only with the chat (not in the tiny loader), so page speed is unchanged.
- Works with page caching (form state in the browser, saving through the REST API with the widget nonce rules).
- Spam protection: honeypot field (the widget already has one) + rate limit per visitor (existing `RateLimiter`).
- Keyboard and screen-reader friendly: labels, focus moves into the form when it opens, Esc / ✕ closes.
- Works inline (shortcode chat) and floating, desktop and mobile.
- Works on the WWC branch too (only the plugin name differs there) when merged on request.

## 9. Open questions

1. ~~**Default trigger**~~ ANSWERED 2026-11-18: after 2 answers + can't answer + offline handoff.
   Buying words (price, quote, demo…) dropped for now (see section 10).
2. ~~**Required fields**~~ ANSWERED 2026-11-18: owner decides per field in the form builder (section 4.1).
3. ~~**Message field**~~ ANSWERED 2026-11-18: owner adds a Textarea field if wanted (not in the default form).
   Field types confirmed 2026-11-18: Text (input text), Number (input number), Email, Textarea.
4. ~~**Recording off**~~ ANSWERED 2026-11-18: leads are **always saved**, recording on or off. With recording off
   the lead has no linked conversation: no transcript link in the Leads tab, no transcript in the email.
5. ~~**Live chat**~~ ANSWERED 2026-11-18: no automatic form during live chat; the team member can send it with
   an "Ask for contact details" button, which opens the form in the visitor's chat (section 3.5). Offline handoff
   trigger stays (no live chat is running then).
6. ~~**Retention**~~ ANSWERED 2026-11-18: kept until the admin deletes them. No retention setting.
7. ~~**Free vs Pro**~~ ANSWERED 2026-11-18: no Free / Pro split. One plugin, everything included.
8. ~~**Logged-in visitors**~~ ANSWERED 2026-11-18: no pre-fill. Logged-in visitors see empty fields like everyone.

All open questions answered. Next = architecture + implementation plan.

## 10. Not in scope (now)

- CRM / newsletter integrations (Zapier, Mailchimp, FluentCRM, webhook): D4, later.
- More field types: checkbox, select / radio, phone with format check, date, file upload (later).
- Conditional fields (show a field only when another has a value).
- Multi-step conversational forms (bot asks one field at a time).
- Lead scoring.
- Buying-word trigger (show the form early when the question has price, quote, demo, contact, call): dropped
  2026-11-18, maybe later.
