# Proposal 1: Self-Hosted Plugin with Paid License

**Model:** The customer installs the plugin, uses their own AI account (API key), and buys a license from us.
**Draft:** 2026-11-09

---

## 1. Summary

We keep the current plugin design and grow it into a complete commercial product. Everything runs on the
customer's own WordPress site. The customer connects their own AI account (OpenAI, Anthropic, Google Gemini,
OpenRouter, or a local model) and deals with that provider directly. A license from us unlocks Pro features,
updates and support.

This is how the market leaders on WordPress work (AI Engine, MxChat, WPBot, AI Puffer).

## 2. How it works

```
Visitor ── question ──► Customer's WordPress site (our plugin)
                         │  1. finds matching content in the site's own knowledge base
                         │  2. sends question + matching content to the AI
                         ▼
                        Customer's AI provider (their own API key)
                         │
                         ▼
Visitor ◄── answer with source links ── WordPress site
```

- **Knowledge base:** stored in the customer's own WordPress database. No external database is needed.
- **Indexing:** done on the admin's computer (local model) or through the customer's AI provider.
- **AI answers:** go from the customer's site to the customer's AI provider, using the customer's key.
- **Our servers:** used only for license checks and plugin updates. No visitor or content data passes through us.

## 3. What the customer gets

### Free version (on wordpress.org)
- AI chatbot that answers only from the site's content, shows source links, and says "I don't know" when the answer isn't there
- Knowledge base from posts, pages, custom post types (docs, FAQs), PDF, Word, text files, and manual Q&A entries
- Content stays in sync automatically when pages are edited, published or deleted
- Choice of AI provider: OpenAI, Anthropic, Google Gemini, OpenRouter, or any compatible provider (including local models)
- Multilingual knowledge base option
- "Can't answer" options: contact page, email, phone or WhatsApp buttons
- 👍/👎 feedback on answers, conversation log, list of unanswered questions with a "fix this answer" button
- Usage tracking plus daily and monthly limits, so AI usage never runs out of control
- Display rules (which pages, devices and users see the chat), accessible widget, style editor with live preview
- Privacy tools: data export and erase, consent notice, no IP addresses stored

### Pro version (licensed)
- More content sources: WooCommerce products, ACF and custom fields, page builders, external URLs and sitemaps
- Members-only knowledge: some content answered only to logged-in users or specific roles (membership and LMS sites)
- WPML / Polylang: separate knowledge base per language
- Lead capture form in chat, team alerts (email, Slack, Telegram, webhook), CRM and Zapier connections
- Full analytics, a content-gap report (topics visitors ask about that the site doesn't cover), and a weekly email digest
- Answer test suite: saved test questions are re-checked after content or setting changes
- Remove "powered by" branding, white label for agencies, multiple chatbots per site
- Priority support

## 4. What we need to build

| Area | Work |
|---|---|
| Plugin features | Must-have list from our research (background auto-sync, more AI providers, multilingual model, more content sources, "can't answer" options, feedback, usage limits, display rules, accessibility, privacy tools) |
| Pro add-on | Separate Pro plugin that adds the paid features to the free plugin |
| Licensing | License keys, site activation, automatic updates (Freemius, Lemon Squeezy, or Easy Digital Downloads) |
| Store | Pricing page, checkout, invoices, EU VAT (handled by the payment platform) |
| Quality | Security review, automated tests, compatibility testing with popular caching plugins, firewalls, page builders and themes |
| Launch | Documentation site, demo site with a live chatbot, setup wizard, wordpress.org listing |

## 5. Advantages

- **Lightest to run:** no AI bills and no servers on our side, apart from license and update hosting.
- **Low risk:** heavy traffic on a customer's chatbot affects their AI account, not ours.
- **Strong privacy story:** data goes only from the customer's site to their own AI provider. Easy for EU and regulated customers.
- **Customer control:** they pick the model and the provider. Agencies like this.
- **Fast to market:** builds on the existing, working architecture.
- **Fits wordpress.org well:** a free plugin plus a Pro version is the standard, trusted model.

## 6. Disadvantages and risks

- **Harder setup:** the customer must create an AI account and paste an API key. Non-technical buyers drop off at this step.
- **AI bill worries:** reviews of competing plugins complain about unexpected AI charges and quota errors.
  Our usage limits reduce this but don't remove it.
- **Crowded field:** AI Engine, MxChat and AI Puffer already sell this model. We must win on answer quality,
  source links, honesty ("I don't know") and simplicity.
- **Provider support load:** each provider has its own errors and limits, and customers will ask us about them.
