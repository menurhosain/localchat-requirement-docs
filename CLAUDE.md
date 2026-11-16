# Knowledge Base Chatbot for WordPress — Research

Project memory. Claude loads this file automatically at session start.
Update it as research progresses: decisions, findings, open questions.

## Goal
BUILD A NEW WordPress plugin: chatbot that answers visitor questions
from the site's own data, like a knowledge base. Core requirement only —
other features (lead gen, live chat, Woo, channels) are secondary.
Competitor research = learn what to copy and where the gaps are.
Claude's role: research assistant.

## Status
- 2026-10-27: Project started. Researched 5 competitor plugins.
- 2026-10-27: User confirmed goal = build own plugin. Core = Q&A over site data.
- 2026-10-28: M0 (project setup) coded in ~/Desktop/dev-site/wp/wp-content/plugins/compact-local-chatbot (git init, no commits).
  Step 2 (M1 DB & core) committed f435cd2. Step 3 (M2 Settings) committed 7fe69b1. Step 4 (M3 Add Content) committed ca43d78. Step 5 (M4 pipeline posts, local model) committed ebb0c79.
- 2026-10-29: Step 6 (M5 files: PDF/DOCX/TXT/MD extraction in worker, PDF page numbers) committed 74e7156. See plan M5 status.
- 2026-10-29: Step 7 (M6 OpenAI/API provider) SKIPPED for now (user's call). Next = step 8 (M7 KB management).
- 2026-10-29: Step 8 (M7 KB screen, chunk viewer, re-embed/remove, ContentWatcher) committed a459fc7. Unpublish = PAUSE (status `paused`, chunks kept, not searched; republish resumes; permanent delete removes). URL column removed (schema v2): URL built at runtime via SourceService::url_for().
- 2026-10-29: Committed e8ddb74 (runtime URLs + paused status). Fixed chunk_hash bug (key = EmbeddingModel::hash_key() = id:revision:dtype:pooling:normalize, sent to worker as model.hashKey) committed d214c0f.
- 2026-10-29: Step 9 (M8): security pass DEFERRED (user's call, do later). Don't start it until asked.
  Tests postponed by user ("will write test later") — don't write new tests until asked.
  "Test connection" button = disabled markup; wire up in M4 (local) + M6 (OpenAI), noted in plan.
  Pending: user activates + tests manually. E2E tests DEFERRED (user's call 2026-10-28) — don't write e2e specs until asked.
- 2026-11-09: Phase 1 + 2 working. NEW GOAL: make plugin commercially sellable. Requirements + research written:
  `docs/commercial-requirements.md` (gaps, customer problems, P0/P1/P2 features, Free vs Pro, open questions). Awaiting user's answers.
- 2026-11-09: Two client proposals drafted in `docs/proposals/`: (1) self-hosted + BYO AI key + yearly license,
  (2) managed AI subscription (our AI gateway pays tokens, OpenRouter key removed from plugin, embedding stays local).
  Hybrid (P2 default + BYO key) noted. User's call: NO timeline or money in proposals. proposals.html = both proposals,
  feature table per proposal (P1 Free/Pro, P2 vs P1), competitor matrix.
- 2026-11-09: proposals.html REWRITTEN for non-technical client (user's call): ONE shared list of 12 top-level features,
  per proposal only architecture-specific additions + good/less good, no compare table, no money/timeline.
  NEW confirmed feature: LIVE CHAT TAKEOVER (admin takes over from dashboard, visitor sees admin replies, AI silent
  until "Hand back to AI"). In commercial-requirements.md as D5 (now P0).
- 2026-11-09: NEW confirmed feature: SMART SALES ALERTS (D6): owner sets situations, AI detects them in chat (+ Woo cart/
  order signals), sends email/Telegram/Slack alert with summary + Take over link. Highlighted box in proposals.html
  after takeover; feature 05 now "Made for WooCommerce".
- 2026-11-12: Implementing LIVE CHAT TAKEOVER (D5). Plan drafted `docs/live-takeover-plan.md` (short polling, schema v4
  conversations.mode/agent_id + messages.kind, LiveChatService, /chat/live, admin Live chat tab, milestones L0–L5).
  Awaiting user's review + answers to its open questions before coding.
  User's calls: admins+editors take over (edit_others_posts, editors see Live chat tab only), visitor sees setting
  "Team member name" (default "Team member"), auto hand-back 10 min, agent msgs in AI history, per-tab conversation,
  no email to left visitors, all FREE, idle back-off after 10 min no interaction.
  FUTURE (not now): messages table one row per message (role visitor/ai/agent/event, content, reply_to), schema v5,
  before first public release; described in docs/live-takeover-flow.html section 9. Live chat only when recording ON (option a); ONE team name for whole site. All questions answered, next = L0.

- 2026-11-12: ONE threshold (user's call) committed 543f818: source_min_score + reason no_relevant_sources removed;
  AnswerService::pick(): meaning filter (each chunk ≥ min_score, none → below_threshold) then keyword top-3 add-back;
  keyword-only mode keeps all hits. Setting label "Minimum match score". Old log rows keep the no_relevant_sources label.

- 2026-11-12: L0 committed cd98894 (schema v4): conversations + mode/agent_id/taken_over_at/visitor_seen_at/agent_seen_at,
  messages + kind/user_id; ConversationRepository live methods (find_by_uuid, add_visitor_message, add_agent_message,
  add_event, take_over atomic, hand_back, taken_over_ids, touch_visitor/agent, messages_since, active). Unanswered/totals
  = qa only. Dev DB migrated (backup of v3 tables in session scratchpad). docs.html updated. Next = L1.

- 2026-11-12: L1 committed 2fd5b90 (server side): LiveChatService (rules, auto hand-back 10 min, capability edit_others_posts
  via filter mnaisp_live_chat_capability), LiveChatController (/live/chats, /messages?after&since, /takeover, /reply,
  /handback), /chat/ask human branch (handled:human, no AI, per-minute limit only), POST /chat/live (poll 15 s ai / 3 s
  human, own rate limit 60/min), team_name setting (default "Team member"), recording off → hand_back_all. MessagePresenter
  shared with Conversations viewer. 34 REST checks passed on dev. Deviations in plan section 10b. Next = L2 (widget).

- 2026-11-12: L2 b10eff6 (widget: live.ts state per tab, polls /chat/live 15 s ai / 3 s human / slower closed,
  hidden, idle; agent bubbles + system notes; human-mode send without embedding/AI; unread count on launcher; loader
  mounts chat closed when stored mode human; history pairs include team replies). L3 21941fb (admin Live chat tab
  pages/LiveChat.tsx: list 5 s, thread 2 s, take over / reply / hand back / take over anyway dialog, team name card
  admins only; editors see only this tab; route #/live/<id>; shared components/MessageRow.tsx). L4 ee46cd3 (viewer "Open in
  Live chat", "messages" labels, privacy text, Style preview sample, preview 600 px). Browser-checked with Playwright +
  system Chromium (throwaway scripts, not committed specs). Deviations: plan sections 10b/10c. Next = L5 (polish).

- 2026-11-12: Online fix 7a813b6 (user's call): closed widget checks in every 60 s also in AI mode; loader mounts chat
  closed on every page once the tab chatted (unless live off); Online < 90 s, Left > 3 min (was 60 s / 2 min). Plan 10d.

- 2026-11-12: Ended-chat fix 684616f: open thread whose chat left the list (AI mode, visitor > 10 min) shows banner "no longer
  live", read-only, polling stops, link to #/conversations/<id> (new deep link opens viewer); comes back if visitor returns.
  "Live" is never stored: live = mode human OR visitor_seen_at within 10 min (LiveChatService::ACTIVE_SECONDS).

- 2026-11-12: Visitor name fix b038232: list/thread name = visitor (WP display name or "Guest"; deleted user → Guest);
  badge = who answers (AI / You / other agent). Guest chat gets user_id when visitor logs in mid-chat (on question,
  message to team, or widget check-in; only fills empty user_id, never moves a chat to another account).

## Scope / Questions to Answer
- [x] Pick existing plugin or build own? BUILD OWN.
- [ ] Existing plugins in market: features, pricing, reviews, weaknesses (5 done)
- [~] Architecture options: RAG chosen, bge-small-en-v1.5 via Transformers.js chosen; LLM provider TBD
- [ ] Content sources: posts, pages, docs plugins (BetterDocs, etc.), PDFs, URLs
- [ ] Hosting constraints: shared hosting, PHP-only vs external service
- [ ] Privacy / GDPR, API cost, rate limits
- [ ] Monetization (if building): freemium, SaaS, WordPress.org rules

## Plugins Under Research
User gives plugins one by one. For each, record: name, URL, what it does,
features, pricing, content sources, AI/LLM used, strengths, weaknesses.
Detail per plugin in `plugins/NN-slug.md`. Summary here.

| # | Plugin | Installs | Model | Hosting | KB sources | Standout |
|---|--------|----------|-------|---------|------------|----------|
| 1 | WPBot (QuantumCloud) | 5,000+ | Freemium, Pro paywalls RAG | Self-host + BYO key | Posts/pages/CPT, PDF/CSV/JSON, Sheets | Huge feature suite, omnichannel, agency lifetime 50 sites |
| 2 | Jotform AI Chatbot | 5,000+ | Jotform plans | Jotform SaaS | Auto-crawl + sync, docs, FAQ, catalog | Brand, conversational forms, Woo, EU endpoint |
| 3 | AI BotKit (WisdmLabs) | 100+ | Free + Pro | Their servers (unclear) | Posts/pages, PDF, URLs, shared KB | Handover w/ chat summary to 5 channels |
| 4 | WebChatAgent | 300+ | Free + paid SaaS | SaaS, no key needed | Pages (JS crawl), PDF, Word, sheets, Notion, Confluence | Live-chat inbox AI<->human, order lookup |
| 5 | AI Chatbot Free Models (NewCodeByte) | 100+ | 100% free, BYO key | Self-host | Posts/pages via SQL keyword only | Free OpenRouter models, 10+ providers |
| 6 | AI Engine (Meow Apps) | 90,000+ | Free; KB/RAG = Pro | Self-host, BYO key; external vector DB (Pinecone/Qdrant/Chroma/OpenAI) | Posts sync, PDF | Market leader, MCP server, every provider |
| 7 | BetterDocs (WPDeveloper) | 30,000+ | Docs plugin; AI chatbot = Pro add-on | BYO key | Docs (method undocumented) | Big docs/KB builder; chatbot opaque |
| 8 | DocsBot | 10+ | DocsBot.ai SaaS plans | SaaS, account | On SaaS side | Thin connector to known SaaS |
| 9 | MxChat | 2,000+ | Free core, paid add-ons | Self-host, LOCAL vector in WP DB, BYO key | Posts/pages/CPT/Woo, PDF, Word, sitemap, URL, YouTube, Q&A; auto-sync | CLOSEST competitor to our goal |
| 10 | AI Puffer (ex AI Power) | 10,000+ | Free + paid + cloud credits | Local WP DB or Pinecone/Qdrant/etc | Posts/pages/CPT/Q&A free; PDF/URL paid; no real auto-sync | Big AI suite, MCP |
| 11 | Helpie FAQ | 9,000+ | Free + Pro | Self-host | n/a | NO AI — FAQ display only; integration target |

## Decisions
- 2026-10-27: Research method = study existing plugins one by one (features + what they do).
- 2026-10-27: Build own plugin. Focus = answering from site data (knowledge base).
- 2026-10-27: Pitch pillars: focused KB-only, data on own server, choose AI provider, free meaning-search w/o external DB,
  auto-sync, visitor-facing citations, "I don't know" fallback, own code/roadmap. "Why build our own" table in plugin-comparison.html.
- 2026-10-28: Approach = RAG (not full content in system prompt). Chunk -> embed -> store in WP DB table -> per question
  retrieve top chunks (semantic + keyword) -> LLM answers only from them w/ citations; low relevance = "I don't know".
- 2026-10-28: Embedding model = bge-small-en-v1.5 (384 dims, English only). Chunks embedded on local machine with
  Transformers.js. User questions embedded by an external Node server (same model, same settings) for search.
  Chat/answer step still uses an LLM provider (TBD).

- 2026-10-28: PHASES. Phase 1 = admin-side indexing only (select content, embed, store, view, re-embed, settings).
  Phase 2 = visitor question embedding (browser vs own Node server, undecided) + search + answers.
- 2026-10-28: Name "Compact Local Chat Bot & Knowledgebase", slug compact-local-chatbot, prefix mnaisp, namespace Mnaisp\, methods unprefixed, text domain mnaisp_translation. Tests: PHPUnit; Playwright unit (DOM-free modules, no browser) + Playwright e2e headless Chromium (wp-env). Frontend: own react+react-dom bundle via Vite, NOT WP React/@wordpress/* packages. UI = shadcn/ui (Radix + Tailwind), preflight scoped to #mnaisp-root, Radix portals into root.
- 2026-10-28: TypeScript for all JS. Tests use existing dev site (docker compose ~/Desktop/dev-site), NOT wp-env;
  PHPUnit integration shares dev DB with prefix wptests_ (wp-phpunit package). Tailwind scoping = custom PostCSS plugin
  (prefix #mnaisp-root + unwrap @layer). Portals render into #mnaisp-portal inside root.
- 2026-10-28: Architecture agreed: layered modular (REST Controllers -> Services -> Repositories), React admin app,
  Strategy for embedding providers (local browser + API via PHP proxy), resource state machine, browser Web Worker
  as pipeline worker, WP hooks for change detection. No pure MVC / event bus. Full plan: `docs/phase-1-plan.md`.

- 2026-10-29: PHASE 2 plan written: `docs/phase-2-plan.md`. Question embedded in VISITOR BROWSER (Transformers.js, same
  model/settings as index); Node server = future feature. Vector search v1 = raw PHP dot product (PhpBruteForceStore)
  behind VectorStoreInterface (SearchService API stable; MariaDB vector / Redis Stack / Qdrant later = derived index,
  MySQL chunks = source of truth, IndexSync via actions). Hybrid with MySQL FULLTEXT + RRF. "I don't know" gate before
  LLM. Chat LLM via ChatProviderInterface (OpenAI-compatible first). Milestones P0–P6.
- 2026-11-03: M6 done with OPENROUTER instead of OpenAI (user's call) committed 809ad54. OpenAIProvider removed.
  OpenRouterProvider (RemoteEmbeddingProviderInterface) uses the Answers key + base URL (user's call: one key). Live model list
  GET /settings/embedding-models (user's call), dimensions detected by a test call on save -> mnaisp_settings.model_info.
  Worker still chunks in browser; vectors via POST /jobs/embed (max 32 texts). Visitor/Playground questions for an API index
  embedded on the server in SearchService (user's call), keyword fallback on failure; SearchQuery.embed=false = keywords only.
  Answer payload has `mode`. Tested live on dev (then restored backup): text-embedding-3-small 1536 dims, question ~1.2 s,
  scores lower than bge (0.62 vs ~0.79) -> thresholds need retuning per model. Free NVIDIA nemotron embed endpoints fail on
  batches >1 (HTTP 500): provider falls back to one text per call on 5xx and remembers it 24 h; API worker batches = 8.
- 2026-11-04: master merged into wwc-group-branding (30a4a43, clean) + 73d580d (privacy-policy text uses WWC name). New user-facing
  plugin-name strings on master need the WWC name on that branch after each merge (git grep "Compact Local Chat Bot").
- 2026-11-04: Empty-answer fix committed abeca45: reasoning model (nemotron free) spent all max_tokens thinking -> empty "answered";
  now ChatProviderException (finish_reason length explained). Max tokens cap 8000, default 2000 (was 600) committed 4108412.
  History messages cut to 4000 chars (server + widget) instead of 400. Dev site still has max_tokens 600 saved.
- 2026-11-04: Conversation log (P5, extended) committed 22b2ecf. User's calls: recording ON by default, retention 90 days,
  store page URL + logged-in user only (no IP). Schema v3: wp_mnaisp_conversations (uuid from widget) + wp_mnaisp_messages
  (row saved "pending" before the AI, completed via AnswerResponder on_finish; stopped = stays pending). Option
  mnaisp_conversation_settings {enabled, retention_days}. Logged-in user read via wp_validate_auth_cookie (widget sends no nonce).
  Daily cron mnaisp_purge_conversations. Admin tab #/conversations: list, Unanswered questions, viewer drawer, settings card.
  Privacy-policy text via wp_add_privacy_policy_content. Not yet: GDPR exporter/eraser, 👍/👎, CSV export.
- 2026-11-04: Select all committed d9fe060: Gmail-style SelectAllBanner (components/SelectAllBanner.tsx) on Add content + Knowledge base.
  Selection = ids, or {all matching filter, exclude[]}; nothing written until the action. Add content: GET /sources returns `available`
  (not-added count, WP_Query + LEFT JOIN resources via scoped posts_join/where filters, same search), POST /resources/add-all
  {type, search, exclude source IDs} -> batches of 500, INSERT IGNORE (ResourceRepository::add_many), confirm dialog.
  KB: POST /resources/reindex and /resources/delete accept {filter:{status,type,search}, exclude ids}.
- 2026-11-03: Branch wwc-group-branding (from master): plugin renamed "AI Chatbot By WWC Group", author WWC Group, menu "AI Chatbot".
  Slug/folder, mnaisp prefix, namespace, text domain, composer package name UNCHANGED. master keeps old name.
- 2026-11-03: Style tab committed 631b576: admin page #/style (pages/Style.tsx) = main colour, position, colors, text_styles in
  grouped cards + live preview (widget/Preview.tsx, real widget CSS in Shadow DOM). Settings → Chat widget = texts/behaviour/limits only.
  Shared widget bits: widget/parts.tsx (Bubble, noteWithLinks), widget/styles.ts (css, registerProperties). Preview launcher duplicates loader.ts look.
- 2026-11-03: Widget colours committed 3fd4bcb: setting colors {header, background, input_area, border, send,
  visitor_bubble, bot_bubble, launcher: #hex | ''}, sent as widget.colors, vars --mnaisp-c-<key> (_ as -), classes .mnaisp-c-*.
  '' = default (main colour or fixed). Border default now #e5e5e5 (was Tailwind v4 currentColor = near black, a bug).
- 2026-11-03: Text styles + px committed b867df5: widget setting text_styles {key: {size px 10–32 | 0, color #hex | ''}} for
  title, bot, visitor, suggestions, input, send, sources, notes, footer, launcher; sent as widget.textStyles, widget sets
  --mnaisp-<key>-size/-color on #mnaisp-root, fallbacks in widget.css (.mnaisp-t-*). Widget CSS = px only (user's call: themes
  with small html font-size broke rem): tools/postcss-rem-to-px.js converts all rem (incl. Tailwind) in widget.css only; admin keeps rem.
- 2026-11-03: "Smart search on visitor devices" no_mobile_data CHANGED (user's call, after iPhone tab crashes ~60 MB model+ORT
  jsep wasm): saveData or cellular = keywords; type known (wifi/ethernet) = download; otherwise download only on DESKTOP
  (userAgentData.mobile===false, else UA without Android/iPhone/iPad/Mobile and not Mac+touch=iPad). Desktop Safari/Firefox
  download (user's call: desktops have the memory); iPhone/iPad/Firefox Android = keywords. Now DEFAULT for new installs.
  Committed 064c8be.
- 2026-11-03: Footer note committed 94fce5c: widget setting footer_note (default old fixed text, ≤300 chars, '' = none), sent as
  widget.footerNote; links as [text](https://…) only (PHP rejects non-http(s), no HTML; widget renders <a target=_blank rel=noopener>).
- 2026-11-03: Launcher label committed e748a2f: widget setting launcher_label (default "Ask AI", ≤30 chars, '' = none), sent as
  widget.launcherLabel in /chat/config; pill above the floating button, click opens chat, hidden while open, aria-hidden.
- 2026-11-02: Suggested questions committed 82b3bcf: widget setting suggested_questions (max 6, ≤150 chars, one per line in admin), sent as widget.suggestions in /chat/config; shown as buttons until the first message (again after ↺). Welcome message now optional.
- 2026-11-02: Widget Stop button committed acaaecc: Send becomes Stop while busy; AbortController aborts fetch/stream (and model wait); entry state "stopped" keeps partial text, never sent as history; ↺ also stops. Enter while busy no longer clears the input.
- 2026-11-02: Widget Markdown: keep the small own renderer (paragraphs, lists, **bold**, [n]); no extension, no react-markdown (user's call).
- 2026-11-02: Conversation memory setting committed 1d5d3ee: history_messages in mnaisp_chat_settings, default 10 (user's call), 0–20, Settings → Answers.
  Widget gets it via /chat/config (fallback 10), REST cap maxItems 20, PromptBuilder keeps last N and starts history with a question.
  Widget now sends only finished question+answer pairs (failed answers left out).
- 2026-11-02: Whole-page + chunk fixes committed 2b848af. Found via "What industry do you server?": Industries page chunk "Dev" (stray Elementor
  text) scored 0.676 on title alone; 11 industry sections 0.43–0.61, list never sent. Fix A (answer time): PageExpander sends whole page
  when page ≤ 4000 chars (filter mnaisp_whole_page_max_chars), sentSources {n, resourceId, chunkIds, wholePage} replaces sentChunkIds.
  Fix B (chunker, TEXT_FORMAT=3): merge chunks < 5 words into neighbour; outline chunk "Title: H1, H2…" for pages with 3+ headings
  (title prefix scored best: 0.79 vs 0.71). User's dev settings: top_k 2, thresholds 0.50.
- 2026-11-02: Editable prompt committed c4959fc: Settings → Prompt card, prompt_template in mnaisp_chat_settings via GET/POST /chat-settings/prompt.
  Placeholders {sources} (required), {dont_know_text}, {site_name}; '' = PromptBuilder::DEFAULT_TEMPLATE. Warnings (not blocking) when
  {dont_know_text} or SAFETY_RULE missing. extra_instructions removed; old value migrated into the template on read. Question + history stay separate messages.
- 2026-11-02: Playground "Generate prompt" (AnswerService::prompt(), POST /playground/prompt; prompt to copy into other chat apps, no AI call) committed ddef854.
- 2026-11-02: Table-aware text committed b465888: tables -> one line per row "Label: value; ..." (PHP ContentExtractor + TS htmlToText, same output);
  chunker splits/overlaps/halves on whole lines (rows, list items). ContentExtractor::TEXT_FORMAT=2 in index meta -> all indexed marked outdated once.
  Duplicate sources: KEEP (user's call, no dedupe).
- 2026-10-29: P4 widget (floating + [mnaisp_chat] shortcode, Shadow DOM, public /chat/ask, rate limits) committed c9680dc.
- 2026-10-29: Streaming (cURL SSE provider, EventStream, /playground/ask-stream, api/stream.ts) committed da4352e.
- 2026-10-29: P3 answers via OpenRouter committed 3628f59 (non-streaming). Streaming = next.
  Source filter added: source_min_score 0.60 (user said "floor threshold 6", read as 0.60) or keyword rank ≤ 3; gate min_score default also 0.60 (user's call; both thresholds 0.60).
  Keyword hits: keep as is (keyword top 3 sent without meaning score; no switch, no extra scoring) — user's call 2026-10-29.
  Idea parked: score keyword-only hits with the question vector + switch "send top keyword matches below threshold".
- 2026-10-29: P2 Playground (search only) committed e3bd7f2.
- 2026-10-29: P1 search core committed a499d3a. Model filter = option B (by model id only, user's call). 10k chunks ≈ 106 ms PHP.
- 2026-10-29: P0 committed 87e5cb6 (events via Transaction::after_commit, shared embedder + embedQuery, GET /chat/config).

## Open Questions (architecture)
- 2026-10-28: Model revision PINNED to HF commit ea104dacec62c0de699686887e3f920caeb4f3e3 (LocalBrowserProvider).
  Index meta change (revision/dtype/model) → admin_init marks all indexed outdated.
- FUTURE TODO (2026-10-28): self-host model files (config, tokenizer, onnx q8) on own server at pinned commit; HF = fallback. See plan section 14.
- Chunks embedded on local machine = manual re-index on content change; conflicts with auto-sync pillar.
  Option: Node server also embeds chunks on post save.
- Node server: who hosts it, auth (API key), rate limit, latency.
- Must match on both sides: model id, dtype/quantization, CLS pooling + normalize, query prefix for bge.
- Chat LLM provider choice. PDFs in v1 or posts/pages only?
- Files (2026-10-28): yes possible — extract text first, then same chunk+embed. Node libs: PDF `unpdf`/`pdfjs-dist`
  (keep page numbers for citation links `file.pdf#page=N`), DOCX `mammoth`, TXT/MD direct. Old .doc + scanned PDFs
  (need OCR, tesseract.js) = skip in v1. Suggest extraction on Node server; WP sends file on Media Library upload.

## Key Findings
- Two architectures: self-hosted + bring-your-own API key (WPBot, Free Models) vs SaaS connector (Jotform, WebChatAgent, likely BotKit).
- Common features: lead capture, human handoff, WooCommerce order/product Q&A, multilingual, proactive triggers.
- Retrieval quality varies: real RAG/vector (WPBot Pro, WebChatAgent) vs keyword SQL (Free Models).
- Auto-sync of content changes = differentiator (Jotform, BotKit).
- Most new entrants (2025) have tiny install bases + few reviews — market still open.
- Big players (AI Engine, AI Puffer) = general AI suites; KB is one feature, often Pro or needs external vector DB.
- MxChat already offers free local vector storage + auto-sync + many sources -> "free local RAG" alone NOT a unique gap anymore.
- Remaining gaps (as of 2026-10-27, 11 plugins): visitor-facing citations (nobody), strict answer-only-from-content + "I don't know" (AI Engine & Puffer admit bot drifts to general knowledge), answer-quality testing tools for admin, focused simple KB-only product, AI layer for docs plugins (BetterDocs chatbot opaque, Helpie has no AI).

## Sources
- wordpress.org plugin pages, fetched 2026-10-27 (URLs in each `plugins/*.md`).

## Files in This Dir
- `CLAUDE.md` — this memory file
- `plugins/` — one detailed note per researched plugin
- `docs.html` — plain-language doc of DB schema (resources, chunks, conversations, messages tables); update when schema changes
- `plugin-comparison.html` — plain-language comparison page (update when new plugins added)
- Plugin code: `~/Desktop/dev-site/wp/wp-content/plugins/compact-local-chatbot/` (README.md there)
- Dev site: http://localhost (nginx + local MariaDB, NOT the docker compose in ~/Desktop/dev-site). WP-CLI works: `cd ~/Desktop/dev-site/wp && wp eval-file x.php --user=1`
  (REST via rest_do_request). Questions can be embedded in Node with @huggingface/transformers from the plugin dir (same model/revision/q8/cls).
- `docs/phase-1-plan.md` — phase 1 (indexing) plan: architecture, DB schema, REST API, pipeline, UI, milestones M0–M8
- `docs/phase-2-plan.md` — phase 2 (search + answers + widget) plan, milestones P0–P6
- `docs/source-selection.md` — how chunks are chosen and sent to the AI (gate, filter, whole page, prompt), worked example; update when AnswerService/PageExpander change
- `docs/chunking.md` — how content becomes chunks + vectors (extract, chunker rules, hash reuse, embed, store, example); update when chunker.ts / ContentExtractor change
- `docs/commercial-requirements.md` — what to add to sell commercially: gaps vs competitors, forum/review pain points, prioritized requirements, release plan
- `docs/proposals/` — client proposals: proposal-1-self-hosted-license.md, proposal-2-managed-ai-subscription.md, proposals.html (client page)
- `docs/live-takeover-plan.md` — live chat takeover (D5) plan: polling, schema v4, REST, widget, admin tab, milestones L0–L5
- `docs/live-takeover-flow.html` — visual flow of a takeover chat (swimlanes, message decision, modes, polling, DB rows, edge cases); update with the plan
- `docs/search-flow.md` — how a question finds its answer: flow, dot product + RRF math, glossary (study doc); update when search changes
- `docs/conversation-search.html` — how follow-ups ("yes", "specification?") are searched: context text, two searches + take-turns merge, clarifying-question + price/quote prompt rules, duplicate-source drop, why, code map, debug query; update when contextQuery / AnswerService merge change
