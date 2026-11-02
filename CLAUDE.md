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
- `docs.html` — plain-language doc of DB schema (resources + chunks tables); update when schema changes
- `plugin-comparison.html` — plain-language comparison page (update when new plugins added)
- Plugin code: `~/Desktop/dev-site/wp/wp-content/plugins/compact-local-chatbot/` (README.md there)
- Dev site: http://localhost (nginx + local MariaDB, NOT the docker compose in ~/Desktop/dev-site). WP-CLI works: `cd ~/Desktop/dev-site/wp && wp eval-file x.php --user=1`
  (REST via rest_do_request). Questions can be embedded in Node with @huggingface/transformers from the plugin dir (same model/revision/q8/cls).
- `docs/phase-1-plan.md` — phase 1 (indexing) plan: architecture, DB schema, REST API, pipeline, UI, milestones M0–M8
- `docs/phase-2-plan.md` — phase 2 (search + answers + widget) plan, milestones P0–P6
- `docs/source-selection.md` — how chunks are chosen and sent to the AI (gate, filter, whole page, prompt), worked example; update when AnswerService/PageExpander change
- `docs/search-flow.md` — how a question finds its answer: flow, dot product + RRF math, glossary (study doc); update when search changes
