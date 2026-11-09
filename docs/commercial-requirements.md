# Commercial Requirements — what to add so the plugin can sell

Written 2026-11-09. Input: 11 competitor notes (`plugins/*.md`), plugin code as of 6d01526,
wordpress.org support forums + 1-star reviews (MxChat, AI Engine, WPBot, AI Puffer), 2026 SaaS chatbot guides.

## 1. Where we stand

### Have today (phase 1 + 2)
- Index posts, pages, PDF/DOCX/TXT/MD; tables as labeled rows; whole short pages; chunk viewer, re-embed, pause on unpublish
- Free local embedding (bge-small, browser) or OpenRouter embeddings
- Hybrid search (vector + FULLTEXT + RRF), "I don't know" gate, source filter, visitor-facing citations
- Answers via OpenRouter, streaming, Stop, editable prompt template, conversation memory
- Widget: floating + shortcode, Shadow DOM, Style tab with live preview, suggestions, launcher label, footer note, mobile-data guard
- Playground (search + "Generate prompt"), Conversations log + Unanswered questions, retention, privacy-policy text

### Our real edge (keep and market)
1. Citations shown to the visitor — no competitor does it well.
2. Strict "answer only from your content" + honest "I don't know" — AI Engine and AI Puffer admit their bots drift.
3. No external vector DB, no SaaS account, data stays on the site.
4. Free meaning-search option (local model, no embedding cost).
5. Playground to test answers before visitors see them.

### Why it can't compete yet (blocking gaps)
| Gap | Why buyer cares | Competitor that has it |
|---|---|---|
| Auto-sync is manual — edits only mark items "outdated", admin must open the admin page so the browser re-embeds | "Set and forget" is the #1 promise of every KB bot. Stale answers = wrong answers | MxChat, Jotform, BotKit (auto) |
| One AI provider (OpenRouter) | Buyers already hold an OpenAI / Anthropic / Gemini key, or want local Ollama | Every competitor: 5–10+ providers |
| English-only local model | Large share of WP sites are non-English; bge-small-en fails them | BotKit 50+ languages, Jotform, WebChatAgent |
| Only posts / pages / files | Docs plugins, FAQ plugins, Woo products, CPTs, ACF fields, external URLs are where the answers live | MxChat, AI Puffer, WPBot |
| No way out when the bot can't answer | "I don't know" with no next step = lost visitor/lead | BotKit handover, WPBot, WebChatAgent, MxChat Slack |
| No cost / abuse controls beyond rate limit | Fear of surprise API bills is a top 1-star theme (AI Engine "unexpected charges", "quota exceeded") | AI Engine limits, Chatbase credits |
| Security pass, tests, e2e deferred | 1-star reviews on WPBot/AI Puffer = "crashed my site", "security issues", "not working" | — |

## 2. Customer problems found (research)

From support forums and 1-star reviews of the closest competitors:

- **Breaks the site / doesn't work** (WPBot, AI Puffer 1-stars): crashes on setup, fatal errors, caching/CSS conflicts, Wordfence 403s (MxChat, AI Engine).
- **Surprise API cost / quota** (AI Engine 1-stars, WPBot "credit exhaustion = bot hangs"). Chat stays active after usage limit (AI Engine forum).
- **Content not indexed**: ACF fields, theme HTML blocks (WoodMart), page builders, SharePoint (MxChat forum).
- **Want auto-sync but with control**: "Manual post selection + auto-sync", "async auto index", "exclude a PAGE from KB" (MxChat forum).
- **Wrong / invented answers** (AI Engine "fictitious texts"; general complaint in 2026 guides — usually a stale or weak KB).
- **Privacy gaps**: update deleted all logs (AI Engine), ratings survive deletion and privacy eraser (MxChat), session id before GDPR consent (AI Engine), retention questions.
- **Access**: role restriction doesn't support custom roles; logged-in / group restriction (both forums).
- **Widget UX bugs**: mobile keyboard pops open after reply, stale reply after "new chat", sticky icon clashes with theme.
- **Free version useless / prices rise, no support** (AI Puffer 1-stars) — pricing trust matters.

From 2026 support-bot guides: buyers judge bots by **resolution rate**, **unanswered topics / content gaps**,
**handoff with full transcript**, **CSAT trend**. Customers (62% in one survey) dislike AI support — so honest answers
+ an easy path to a human matter more than more AI features.

## 3. Requirements

Priority: **P0** = before first paid release. **P1** = first updates after launch. **P2** = later / differentiation.
Tier: **Free** (wordpress.org) / **Pro** (paid).

### A. Keep the knowledge base fresh (robustness)
| ID | Requirement | Pri | Tier |
|---|---|---|---|
| A1 | **Background auto-sync**: changed/new content re-embedded without an admin browser. API embedding: PHP chunker + WP-Cron / Action Scheduler queue. Local model: re-embed runs on next admin page load (any admin screen, small background worker) + clear "N items waiting" notice | P0 | Free |
| A2 | Per-type auto-add rules: "auto-add new posts of type X / category Y" (opt-in) and exclude list (page/post/category/tag) | P0 | Free |
| A3 | Content source health: items failing extraction, empty text, too short — shown with reason | P1 | Free |
| A4 | Scheduled full re-check (daily cron compares content hash) as safety net for missed hooks | P1 | Free |
| A5 | Port chunker to PHP (same rules as chunker.ts, shared fixture tests) so server can index without browser | P0 | Free |

### B. More content sources
| ID | Requirement | Pri | Tier |
|---|---|---|---|
| B1 | Any public custom post type (docs, FAQ, products, courses) | P0 | Free |
| B2 | Manual Q&A / snippets ("Q: return policy? A: …") — fastest fix for a bad answer | P0 | Free |
| B3 | Page builders + custom fields: Elementor/Divi/Bricks rendered output, ACF/Meta Box fields picker | P1 | Pro (ACF picker) |
| B4 | WooCommerce products (title, price, stock, attributes, short desc) as structured text | P1 | Pro |
| B5 | Docs/FAQ plugin presets: BetterDocs, Helpie FAQ, Echo KB, weDocs, Easy Accordion | P1 | Free (marketing hook) |
| B6 | External URL + sitemap import, periodic refresh | P1 | Pro |
| B7 | CSV/XLSX, HTML files; Google Docs/Sheets, Notion | P2 | Pro |
| B8 | Scanned PDF OCR | P2 | Pro |

### C. AI providers and models
| ID | Requirement | Pri | Tier |
|---|---|---|---|
| C1 | Chat providers: OpenAI, Anthropic, Google Gemini, OpenRouter, any OpenAI-compatible URL (Ollama, LM Studio, Groq, Mistral, DeepSeek) | P0 | Free |
| C2 | Embedding providers: OpenAI, Gemini, Voyage/Cohere, OpenAI-compatible, OpenRouter, local | P1 | Free |
| C3 | **Multilingual local model** (e.g. multilingual-e5-small / bge-m3 small q8) as option; answer in visitor's language | P0 | Free |
| C4 | WPML / Polylang: index per language, search only visitor's language | P1 | Pro |
| C5 | Fallback provider if main one fails (e.g. OpenAI down → OpenRouter) | P2 | Pro |
| C6 | Per-model threshold presets (bge 0.60 vs text-embedding-3-small ~0.45) + "auto-tune from Playground" | P1 | Free |

### D. When the bot can't answer (handoff + leads)
| ID | Requirement | Pri | Tier |
|---|---|---|---|
| D1 | "I don't know" next step: configurable buttons — contact page link, email, phone, WhatsApp | P0 | Free |
| D2 | In-chat contact form (name, email, message) → email to admin with transcript; stored as lead | P1 | Free basic / Pro fields |
| D3 | Notify team on unanswered/handoff: email, Slack, Telegram, webhook | P1 | Pro |
| D4 | Lead export CSV + webhook / Zapier / FluentCRM / Mailchimp | P2 | Pro |
| D5 | **Live chat takeover** (confirmed 2026-11-09, headline feature): admin sees live chats, clicks Take over → visitor sees "team member joined", admin replies appear in widget, AI does NOT answer while taken over; Hand back to AI resumes bot. Needs live channel (polling/SSE) both ways + conversation state `ai`/`human` | P0 | Pro or all plans (TBD) |
| D6 | **Smart sales alerts** (confirmed 2026-11-09, headline feature, esp. WooCommerce): owner defines situations in plain words (bulk order/quote, discount ask, wants a human, unhappy, 2× "I don't know"; Woo: big cart no checkout, repeat product questions, out-of-stock ask, stuck at checkout, returning customer). AI classifies each conversation turn against them (small extra LLM call or structured output with the answer) + Woo signals (cart, orders, viewed product) → alert via email / Telegram / Slack with AI summary, visitor context, quote, Take over link (D5). Needs dedupe (one alert per conversation per situation) | P0 | Pro (TBD) |

### E. Answer quality loop (our differentiator — "test and improve")
| ID | Requirement | Pri | Tier |
|---|---|---|---|
| E1 | 👍/👎 on each answer + optional comment; stored on message | P0 | Free |
| E2 | "Fix this answer" from Conversations/Unanswered: create Q&A snippet (B2) or link to page to index | P0 | Free |
| E3 | Analytics: questions/day, answered vs "I don't know" rate, 👍 rate, top questions, top sources, top unanswered | P1 | Free basic / Pro full |
| E4 | Content-gap report: group unanswered questions by topic (embedding clustering, no AI cost) | P1 | Pro |
| E5 | **Test suite**: saved questions with expected source/answer; re-run after content/model/prompt change, show pass/fail | P1 | Pro |
| E6 | Weekly email digest: volume, answer rate, new gaps | P2 | Pro |
| E7 | Admin "why this answer" view: chunks, scores, prompt (exists in Playground — link from Conversations) | P1 | Free |

### F. Cost, abuse and safety
| ID | Requirement | Pri | Tier |
|---|---|---|---|
| F1 | Token/cost tracking per message (usage from provider), monthly totals in admin | P0 | Free |
| F2 | Budget caps: per visitor/day messages, site-wide monthly token/$ cap → graceful "limit reached" + contact options | P0 | Free |
| F3 | Bot protection: honeypot + optional Cloudflare Turnstile; block by IP range/user agent | P1 | Free |
| F4 | Prompt-injection guard: content treated as data, refuse role changes, output links only to own domain (filter) | P0 | Free |
| F5 | Answer cache for repeated identical questions (saves cost, faster) | P2 | Pro |
| F6 | Graceful provider failure: show keyword-search results as links instead of an error | P0 | Free |

### G. Widget and placement
| ID | Requirement | Pri | Tier |
|---|---|---|---|
| G1 | Display rules: per page/post type/URL pattern, logged-in/out, roles, device | P0 | Free |
| G2 | Gutenberg block + Elementor widget for inline chat | P1 | Free |
| G3 | Accessibility: keyboard, focus trap, ARIA live region, contrast check, reduced motion (WCAG 2.2 AA) | P0 | Free |
| G4 | Bot name + avatar, dark mode, full-page mode, open-on-load / after X seconds / scroll | P1 | Free basic / Pro triggers |
| G5 | Remove footer branding / white label admin | P1 | Pro (agency) |
| G6 | Multiple bots: different KB subset + prompt + style per bot (e.g. docs vs sales) | P2 | Pro |
| G7 | Mobile polish: no auto keyboard after reply, safe-area, full-screen on phones | P0 | Free |

### H. Private / member knowledge
| ID | Requirement | Pri | Tier |
|---|---|---|---|
| H1 | Respect post visibility: private/password/members-only content never answered to guests | P0 | Free |
| H2 | Role-scoped KB: some sources only for logged-in roles (membership, LMS, intranet) — MemberPress, LearnDash, Paid Memberships Pro | P1 | Pro |
| H3 | Admin-only assistant in wp-admin over internal docs | P2 | Pro |

### I. Privacy and compliance
| ID | Requirement | Pri | Tier |
|---|---|---|---|
| I1 | WP personal-data exporter + eraser for conversations, leads, ratings (all linked data deleted) | P0 | Free |
| I2 | Consent: optional "start chat = accept" notice; integration with Complianz / CookieYes; no storage before consent | P0 | Free |
| I3 | Never store IP (keep), mask emails/phones in logs option | P1 | Free |
| I4 | Docs: data-flow page (what goes to which provider), DPA template, EU-friendly setup (Mistral/EU endpoints, local embeddings) | P0 | — |

### J. Product robustness (sellable quality)
| ID | Requirement | Pri | Tier |
|---|---|---|---|
| J1 | Security pass (deferred M8): capability checks, nonces, REST permission callbacks, escaping, SQL prepare, upload handling, public endpoint abuse | P0 | — |
| J2 | Tests: PHPUnit for services, Playwright e2e for setup → index → ask flow; CI on PHP 8.1–8.4, WP latest-2 | P0 | — |
| J3 | Compatibility matrix: caching plugins (WP Rocket, LiteSpeed, W3TC), Cloudflare, Wordfence, Elementor, Divi, popular themes; no JS/CSS leak (Shadow DOM helps) | P0 | — |
| J4 | Onboarding wizard: provider + key → test connection → pick content → index → test question → place widget (target < 10 min) | P0 | Free |
| J5 | Site Health checks: FULLTEXT support, PHP memory, cron running, provider reachable, table versions | P1 | Free |
| J6 | Scale: > 50k chunks — MariaDB 11.7 VECTOR or optional Qdrant behind existing VectorStoreInterface; batch limits | P2 | Pro |
| J7 | Settings import/export, safe migrations (never delete logs on update — AI Engine bug), uninstall option "keep data" | P1 | Free |
| J8 | Multisite support (per-site KB) | P2 | Pro |
| J9 | Translations: all strings via text domain, .pot file, RTL widget | P1 | Free |
| J10 | Debug log (provider errors, timings) downloadable for support | P1 | Free |

### K. Business side
| ID | Requirement | Pri |
|---|---|---|
| K1 | Free on wordpress.org (guidelines: no trialware, disclose external services in readme, no locked features in free code) + separate Pro plugin | P0 |
| K2 | Licensing + updates + checkout: Freemius, Lemon Squeezy, or EDD Software Licensing | P0 |
| K3 | Docs site, demo site with live bot answering from the docs, 2-min video | P0 |
| K4 | Pricing: yearly + lifetime; 1 / 5 / unlimited sites; agency tier with white label | P0 |
| K5 | Opt-in usage telemetry (no visitor data) to learn which features are used | P2 |

## 4. Suggested release plan

**v1.0 (sellable minimum)** — all P0:
A1 A2 A5, B1 B2, C1 C3, D1 D5 D6, E1 E2, F1 F2 F4 F6, G1 G3 G7, H1, I1 I2 I4, J1 J2 J3 J4, K1–K4.
Pitch: *"The AI chatbot that answers only from your site, shows its sources, says 'I don't know' and tells you what to add. Stays in sync. Your data, your AI key, no vector DB."*

**v1.1** — fix-the-gaps release: B3 B4 B5 B6, C2 C4 C6, D2 D3, E3 E4 E5 E7, F3, G2 G4 G5, H2, I3, J5 J7 J9 J10.

**v2** — B7 B8, C5, D4, E6, F5, G6, H3, J6 J8, K5.

## 5. Suggested Free vs Pro split

- **Free must be genuinely useful** (AI Puffer 1-stars: "free version is USELESS"): full RAG, posts/pages/CPTs/files,
  citations, I-don't-know + contact buttons, auto-sync, 👍/👎, basic analytics, all major providers, multilingual model.
- **Pro sells time-saving and business outcomes**: Woo/ACF/URL sources, role-scoped KB, WPML, lead form fields + notifications +
  integrations, content-gap report, test suite, digest emails, white label, multiple bots, external vector store.

## 6. Open questions for the user
1. Target buyer first: small business support sites, docs/SaaS product sites, WooCommerce stores, or agencies?
2. Keep "local model in browser" as headline, or make API embeddings default (simplifies auto-sync A1)?
3. Sell on wordpress.org (free + Pro) or Pro-only from own site?
4. ~~Is live human chat (D5) in scope?~~ YES — live chat takeover confirmed 2026-11-09.
5. Brand: "Compact Local Chat Bot" or "AI Chatbot By WWC Group" for the commercial release?
6. Licensing platform preference (Freemius takes ~30% but handles tax/EU VAT; Lemon Squeezy/EDD cheaper, more work)?

## Sources
- wordpress.org support forums: https://wordpress.org/support/plugin/mxchat-basic/ , https://wordpress.org/support/plugin/ai-engine/
- 1-star reviews: https://wordpress.org/support/plugin/chatbot/reviews/?filter=1 , https://wordpress.org/support/plugin/gpt3-ai-content-generator/reviews/?filter=1 , https://wordpress.org/support/plugin/ai-engine/reviews/?filter=1
- Customer view of AI chatbots: https://www.hostingadvice.com/studies/ai-chatbot-customer-satisfaction/amp/
- Support-bot KPIs: https://www.usefini.com/guides/support-chatbot-tools-first-contact-resolution-analytics , https://wonderchat.io/blog/customer-support-kpis-2026
- Chatbase features (SaaS reference): https://www.whisperchat.ai/alternatives/chatbase , https://use-apify.com/docs/apify-vs-the-world/chatbase-review
