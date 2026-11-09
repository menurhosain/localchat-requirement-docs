# Proposal 2: Managed AI Subscription

**Model:** The customer installs the plugin and subscribes to our service. We provide the AI. No API key and no AI account are needed.
**Draft:** 2026-11-09

---

## 1. Summary

The knowledge base and indexing stay on the customer's WordPress site, as today. The **AI answering step moves to
our own servers**. The customer never sees or manages an AI provider or API key. They connect their site to their
account with one click, choose a plan, and the chatbot works.

We hold the AI provider account and include AI usage in the subscription.

## 2. How it works

```
Visitor ── question ──► Customer's WordPress site (our plugin)
                         │  1. finds matching content in the site's own knowledge base
                         │  2. sends question + matching content to our AI service (site token)
                         ▼
                        OUR AI SERVICE
                         │  checks plan + monthly usage, adds our tuned instructions,
                         │  picks the model, sends the request to the AI provider (our key)
                         ▼
                        AI provider (OpenAI / Anthropic / Google / OpenRouter …)
                         │
                         ▼
Visitor ◄── streamed answer with source links ── WordPress site ◄── our AI service
```

- **Knowledge base:** stays in the customer's WordPress database.
- **Indexing:** stays local. Embedding still runs on the admin's computer, as today.
- **Visitor questions:** searched on the customer's site. Only the question, the few matching passages and short chat
  history are sent to our service.
- **AI provider key:** only on our servers. The OpenRouter key field and provider settings are removed from the plugin.
- **Site token:** stored server-side in WordPress, never sent to visitors' browsers.

## 3. What the customer gets

Everything in the Proposal 1 feature list, with these changes:

**Simpler:**
- Setup is install → "Connect site" → choose plan → index content → done (target: under 5 minutes)
- No AI account, no API key, no separate AI provider bill
- We choose and tune the best model and instructions for answering from website content, and improve them over time for every customer
- Automatic fallback: if one AI provider is down, our service switches to another

**New in the plugin:**
- Account screen: connection status, plan, monthly usage meter, upgrade button
- Clear message to visitors when the monthly limit is reached (with contact options), plus an email to the site owner at 80% and 100%
- Admin testing in the Playground has its own allowance, so it doesn't use up visitor answers

**Removed from the plugin:**
- AI provider choice, API key fields, model lists
- Full prompt editing becomes "extra instructions" (our service keeps the core answering and safety rules)

**Customer web dashboard (new, on our side):**
- Plans, invoices, connected sites, usage history

**Possible add-on later:** "Cloud indexing". Our service creates embeddings on the server, so content syncs fully
automatically without the admin's computer, and more languages are supported.

## 4. What we need to build

| Area | Work |
|---|---|
| Plugin features | Same must-have list as Proposal 1, minus the provider and API key settings. Plus: connect-site flow, usage meter, limit messages |
| AI service (new) | API gateway: site authentication, plan and usage limits, rate limiting, our prompt and safety rules on the server, streaming answers, provider fallback, usage metering |
| Accounts and billing (new) | Sign up, connect sites, subscriptions, invoices, EU VAT. Paddle or Lemon Squeezy as merchant of record |
| Customer dashboard (new) | Plans, usage, sites, invoices |
| Operations (new) | Hosting, backups, monitoring and alerts, status page, logs with short retention |
| Legal (new) | Terms of service, privacy policy, Data Processing Agreement (DPA), list of sub-processors (AI providers, hosting) |
| Quality | Security review of plugin **and** service, automated tests, load testing of the service |
| Launch | Documentation, demo site, wordpress.org listing (external service disclosed in the readme, as required) |

## 5. Advantages

- **Easiest setup on the market for a self-hosted knowledge base:** removing the API key step removes the biggest drop-off point.
- **Predictable for customers:** one subscription, no surprise AI bills (the top complaint in competitor reviews).
- **Recurring subscription business** that grows as customers grow.
- **Quality control:** we own the prompt, the model choice and the safety rules, and can improve answer quality for all customers at once.
- **Privacy middle ground:** unlike SaaS chatbots (Jotform, WebChatAgent, DocsBot), the full knowledge base stays on
  the customer's site. Only the passages needed for each answer pass through us.
- **Usage insight** (without storing content) shows which plans and features matter.
- **Clear upgrade path:** "Cloud indexing" and premium models can become higher tiers later.

## 6. Disadvantages and risks

| Risk | Impact | How we reduce it |
|---|---|---|
| We carry the AI usage | Bots or attacks on a customer's chat widget use our AI capacity | Per-site monthly limits, per-visitor rate limits, bot protection, server-side caps on request size |
| Site token misuse (using our service as a free general AI) | Unplanned usage | Service accepts only "question + sources" requests, applies our own instructions, limits length, counts usage against the plan |
| AI provider price or policy changes | Our costs change | Multi-provider routing, ability to switch models |
| Our service goes down | Every customer's chatbot stops | Fallback providers, monitoring, status page. Plugin shows contact options and search links if the service can't be reached |
| We become a data processor | GDPR duties, DPA, possible EU hosting requirement | EU hosting option, no long-term storage of questions or content, providers with zero data retention, signed DPAs |
| More to build and run | More operations work (on-call, billing support) | Managed hosting, merchant-of-record billing, keep the service small |
| Some buyers want their own AI key | "Your data, your key" message gets weaker | Optional "bring your own key" mode for agencies (hybrid, see note below) |

---

### Note: hybrid option

Several competitors (for example AI Puffer: cloud credits **or** your own key) offer both. A later version could
keep Proposal 2 as the default ("works out of the box") and let agency or enterprise customers plug in their own key.
That keeps the easy setup for most buyers and the full-control option for agencies and privacy-sensitive clients.
