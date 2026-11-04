# Phase 2 Plan — Search & Answers (Visitor Chat)

> **Goal:** A visitor asks a question on the site. The question is embedded **in the visitor's browser**, the server
> finds the best matching chunks (meaning + keywords), and an LLM answers **only** from those chunks, with clickable
> citations. When nothing relevant is found, the bot says "I don't know" instead of guessing.
>
> **Builds on phase 1:** chunks + vectors in `wp_mnaisp_chunks`, model settings in `mnaisp_index_meta`,
> `ResourceStatus::searchable()` (indexed + outdated), runtime URLs (`SourceService::url_for()`).
>
> **Not in phase 2:** own Node embedding server (future feature), server-side auto re-embed, lead capture,
> live chat handoff, WooCommerce.

Created: 2026-10-29

---

## 1. Summary

| Item | Decision |
|---|---|
| Question embedding | Visitor's browser: Transformers.js in a Web Worker, same model + settings as the index (from `mnaisp_index_meta`) |
| Vector search (v1) | Raw PHP: dot product over all searchable chunk vectors (vectors are normalized → dot product = cosine similarity) |
| Search design | **Strategy + repository**: `SearchService` talks to a `VectorStoreInterface`; PHP brute force first, MariaDB vector / Redis Stack / Qdrant later, same API |
| Keyword search | MySQL `FULLTEXT(content)` (index already exists), behind `KeywordSearchInterface` |
| Combining results | Reciprocal Rank Fusion (RRF) of semantic + keyword ranks |
| "I don't know" | Gate **before** the LLM: best semantic score below a threshold → fixed answer, no LLM call (no cost) |
| Answer generation | LLM via PHP proxy (key stays on server), `ChatProviderInterface` Strategy; first provider: OpenAI-compatible API |
| Citations | Numbered sources `[1]` → title + runtime URL (+ `#page=N` for PDFs) |
| Visitor UI | Chat widget, own React bundle, rendered in **Shadow DOM** (theme CSS can't break it, ours can't break the theme) |
| Admin tool | **Playground**: ask a question, see ranked chunks with scores, answer and citations (answer-quality testing — a market gap) |
| Source of truth | MySQL chunk table always. Any external vector store is a **derived index** that can be rebuilt from it |

---

## 2. Architecture

```
┌─ Visitor browser ──────────────────────────────────────────────┐
│  Widget (React, Shadow DOM)                                     │
│   question ──▶ Web Worker: Transformers.js bge-small (q8)        │
│                 query prefix + CLS pooling + normalize           │
│                 ──▶ question vector (384 floats)                 │
│   POST /chat/ask { question, vector, modelKey, history }         │
└────────────────────────────┬───────────────────────────────────┘
                             │  WordPress REST API (mnaisp/v1, public)
┌────────────────────────────▼───────────────────────────────────┐
│ Rest/ChatController      rate limit, validate, model key check  │
│ Chat/AnswerService       search → gate → prompt → LLM → cite    │
│ Search/SearchService     semantic + keyword → fuse → load text  │
│   ├── VectorStoreInterface ── PhpBruteForceStore (v1)            │
│   │                          MariaDbVectorStore   (later)        │
│   │                          RedisStackStore      (later)        │
│   │                          QdrantStore          (later)        │
│   └── KeywordSearchInterface ── MySqlFulltextSearch              │
│ Chat/ChatProviderInterface ── OpenAICompatibleProvider (v1)      │
│ Search/IndexSync          keeps external stores in step (events) │
│ Repositories (phase 1)    chunks + resources = source of truth   │
└────────────────────────────────────────────────────────────────┘
```

Flow of one question:
1. Widget gets `/chat/config` once (model id, revision, dtype, pooling, normalize, query prefix, `modelKey`, UI texts).
2. Worker embeds `query_prefix + question` (bge needs the prefix for questions, not for chunks).
3. `POST /chat/ask` with question text, vector, `modelKey`, last few turns.
4. Server: `modelKey` must equal current `EmbeddingModel::hash_key()`; else 409 → widget reloads config + model.
5. `SearchService::search()` → top chunks with scores.
6. Gate: best semantic score < `min_score` → "I don't know" answer (configurable text), no LLM call.
7. `PromptBuilder` puts numbered sources + rules into the prompt; `ChatProvider` streams the answer.
8. Server sends sources (title, URL, page) + answer tokens; widget renders citations.

---

## 3. Search Layer (switchable backend)

### 3.1 Contracts

```php
namespace Mnaisp\Search;

/** What to search for. Immutable. */
final class SearchQuery {
    public function __construct(
        public readonly string $text,           // question text (keyword search, logging)
        public readonly ?array $vector,         // float[384], normalized; null = keyword only
        public readonly string $model_key,      // EmbeddingModel::hash_key() the vector was made with
        public readonly int $limit = 8,
        public readonly SearchFilter $filter = new SearchFilter(),
    ) {}
}

/** Optional narrowing; every store must support these. */
final class SearchFilter {
    public function __construct(
        public readonly ?array $source_types = null,  // ['post'], ['file']
        public readonly ?array $post_types = null,    // ['page', 'docs']
        public readonly ?array $resource_ids = null,
    ) {}
}

/** One hit. Score contract: cosine similarity in [-1, 1], higher = better, for EVERY store. */
final class ScoredChunk {
    public function __construct(
        public readonly int $chunk_id,
        public readonly int $resource_id,
        public readonly float $score,
    ) {}
}

interface VectorStoreInterface {
    public function id(): string;                          // 'php', 'mariadb', 'redis', 'qdrant'
    public function label(): string;
    /** Can it run here (DB version, PHP extension, reachable URL)? Reason when not. */
    public function availability(): true|string;
    /** Top $query->limit chunks of searchable resources, best first. */
    public function search( SearchQuery $query ): array;   // ScoredChunk[]

    // Sync hooks. No-ops for stores that read the chunk table directly (php, mariadb on same table).
    public function upsert_resource( int $resource_id ): void;   // (re)write all its chunks
    public function delete_resource( int $resource_id ): void;
    /** Rebuild from the chunk table in batches; returns progress so the admin can run it step by step. */
    public function rebuild( int $offset, int $batch ): RebuildProgress;
    public function needs_sync(): bool;                    // false for php
}

interface KeywordSearchInterface {
    /** Ranked chunk ids. Scores are NOT comparable with cosine; only the rank is used. */
    public function search( SearchQuery $query ): array;   // ScoredChunk[]
}
```

Rules every store follows:
- **Only searchable resources** (`ResourceStatus::searchable()`: indexed + outdated). Paused, failed, queued never match.
- **Only vectors of the current model key.** Mixed vectors are never compared.
- **Same score scale** (cosine similarity). Stores that return distance convert it (`1 - distance`), so `min_score`
  means the same thing whatever the backend.
- Stores never return text; `SearchService` loads text + resource details from MySQL (one query for the top chunks).

### 3.2 `SearchService` (stable API used by the chat and the Playground)

```php
final class SearchService {
    public function __construct(
        private VectorStoreInterface $vectors,     // from VectorStoreRegistry::active()
        private KeywordSearchInterface $keywords,
        private RankFusion $fusion,
        private ChunkRepository $chunks,
        private ResourceRepository $resources,
        private SourceService $sources,            // runtime URLs
    ) {}

    /** @return SearchResult (hits with text, heading, page, title, url, semantic/keyword/fused scores, timings) */
    public function search( SearchQuery $query ): SearchResult;
}
```

Steps inside: semantic top N (e.g. 20) + keyword top N → RRF (`score = Σ 1 / (60 + rank)`) → top `limit` →
load text → `SearchResult`. Keeps both raw scores for the gate and the Playground.

If the active store throws or is unavailable → fall back to `PhpBruteForceStore`, log, admin notice. The chatbot keeps
working.

### 3.3 `PhpBruteForceStore` (v1)

```
SELECT c.id, c.resource_id, c.embedding
FROM wp_mnaisp_chunks c JOIN wp_mnaisp_resources r ON r.id = c.resource_id
WHERE r.status IN ('indexed','outdated') AND c.embedding_model = %s  [+ filter]
ORDER BY c.id  LIMIT 2000 OFFSET …            -- batches: memory stays flat
```
- Per row: `unpack('g*', $blob)` → 384 floats → dot product with the question vector.
- Keep only the best K in a small min-heap (`SplPriorityQueue`), never all scores.
- Model check: rows whose `embedding_model` differs are skipped (and the index meta check makes this rare).
- Expected cost (PHP 8.x): ~2k chunks ≈ 10–30 ms, ~10k ≈ 50–150 ms. Measured in P6; budget **< 200 ms at 10k chunks**.
- Optional speed-up (only if P6 shows need): cache all packed vectors of the current model in one blob in the object
  cache / a file under `uploads/`, invalidated by `IndexSync` events. Same store class, `upsert/delete` then clear the cache.

### 3.4 Later stores (same contract, switch in Settings)

| Store | When available | How it works | Sync |
|---|---|---|---|
| `MariaDbVectorStore` | MariaDB 11.7+ | Own table `mnaisp_chunk_vectors (chunk_id, resource_id, embedding VECTOR(384), VECTOR INDEX … DISTANCE=cosine)`; `ORDER BY VEC_DISTANCE_COSINE(embedding, VEC_FromText(?)) LIMIT k` | upsert/delete on events (own table: dbDelta and old DBs can't hold VECTOR in the main table) |
| `RedisStackStore` | Redis Stack / RediSearch module (not plain Redis) | `FT.CREATE … VECTOR HNSW … DISTANCE_METRIC COSINE`; `FT.SEARCH … KNN` with tag filters for post type / source type | upsert/delete on events |
| `QdrantStore` | Qdrant URL (+ API key, encrypted) | Collection per site + model key; point id = chunk id; payload: resource_id, post_type, source_type; REST via `wp_remote_post` | upsert/delete on events; collection rebuilt when model key changes |

Plain Redis / Memcached is **not** a vector store; it's only the cache speed-up for the PHP store (3.3).

Switching store: Settings → "Search engine" → availability check → "Build index" runs `rebuild()` in batches with a
progress bar (admin tab, like the embedding queue) → only then switch active store. Until the build finishes, the old
store keeps answering.

### 3.5 `IndexSync` (keeps derived stores correct)

Phase 1 code fires new actions (added in P0):

| Action | Fired from | Store reaction |
|---|---|---|
| `mnaisp_chunks_replaced( $resource_id )` | `ChunkRepository::replace()` after commit | `upsert_resource()` |
| `mnaisp_resource_deleted( $resource_id )` | `ResourceRepository::delete()` | `delete_resource()` |
| `mnaisp_resources_status_changed( ?array $ids, ResourceStatus $to )` | `ResourceRepository::transition()` | becomes searchable → upsert; leaves searchable (paused, queued…) → delete |
| `mnaisp_index_meta_changed()` | `IndexingService::sync_index_meta()` / settings | drop + rebuild (new model key) |

`IndexSync` listens and calls the active store only if `needs_sync()`. Failed sync calls are queued in an option and
retried by WP-Cron; "Rebuild index" always repairs everything.

---

## 4. Question Embedding in the Visitor's Browser

- Reuse phase 1 code: move `local-embedder.ts` + types into `assets/src/shared/embedding/` used by both admin worker
  and widget worker. Add `embedQuery(text)` = `query_prefix + text`, same pooling/normalize.
- Same pinned revision + dtype from `/chat/config` (= `mnaisp_index_meta`), so question and chunk vectors match.
- **Lazy:** nothing loads until the visitor opens the widget (or focuses the input). Page speed is untouched.
- **Size warning (main risk):** first use downloads the model (~34 MB q8) + ONNX Runtime engine (several MB) from
  Hugging Face / jsDelivr; cached afterwards (Cache Storage). Mitigations:
  - start the download when the widget opens, show "Preparing smart search (first time only)…";
  - **keyword-only fallback** while loading, or when the device can't (no WebAssembly, very low memory,
    `navigator.connection.saveData`), or when the admin turns browser embedding off → `vector: null`,
    answer from FULLTEXT results (lower quality, still cited, still gated);
  - admin setting: "Semantic search on visitor devices: on / off / only when not on mobile data".
- Future feature: own Node server embeds questions (no visitor download); only the widget's embed step changes,
  the server API stays the same (`vector` comes from the server instead of the browser).

---

## 5. Answers (LLM)

### 5.1 `ChatProviderInterface` (Strategy, like embedding providers)

```php
interface ChatProviderInterface {
    public function id(): string;
    public function models(): array;
    /** Stream answer tokens to $on_token; returns the full answer + usage. */
    public function complete( ChatRequest $request, callable $on_token ): ChatResponse;
    public function test(): true|string;   // "Test connection"
}
```
v1: `OpenAICompatibleProvider` (base URL + key + model) → covers OpenAI, OpenRouter, Groq, Together, Mistral,
local Ollama / LM Studio. Anthropic / Gemini native later. Key encrypted with existing `Crypto`, shown masked.

### 5.2 Prompt rules (`PromptBuilder`)
- System prompt: answer **only** from the numbered sources; if they don't contain the answer, reply with the exact
  "I don't know" text; cite like `[1]`; no outside knowledge; site name + tone from settings.
- Sources: `[n] Title › Heading (page N)\n chunk text`, best first, within a token budget (e.g. 6 chunks / ~2,500 tokens).
- History: last 3 turns, for wording only. v1 retrieval uses the current question alone (known weakness for
  follow-ups like "and how much is it?"; later: condense question with one cheap LLM call, or embed last 2 user turns).

### 5.3 "I don't know" (two layers)
1. **Before the LLM:** best semantic score < `min_score` (default 0.60, user's call 2026-10-29; tuned in the Playground) and no
   strong keyword hit → fixed text, no LLM cost. Keyword-only mode: no FULLTEXT hit → fixed text.
2. **Inside the LLM:** prompt rule above. The answer is checked for the exact phrase so the widget can show
   "contact us" options.

### 5.4 Citations
Server sends the sources first (`[n]` → title, `SourceService::url_for()` URL, `#page=N` for PDFs, heading). The widget
turns `[n]` in the answer into links and lists "Sources" under it. Only sources the answer actually cites are shown.

### 5.5 Streaming
`POST /chat/ask` answers as Server-Sent Events (`sources`, `token`, `done`, `error`). Some hosts/proxies buffer
(nginx `proxy_buffering`, Cloudflare); send `X-Accel-Buffering: no` and fall back to a normal JSON response when the
first event doesn't arrive in time (setting: streaming on/off).

---

## 6. REST API (`/wp-json/mnaisp/v1`)

| Method | Route | Who | Purpose |
|---|---|---|---|
| GET | `/chat/config` | public | Model settings + `modelKey`, widget texts, whether browser embedding is on |
| POST | `/chat/ask` | public, rate-limited | `{ question, vector?, modelKey, history[] }` → SSE / JSON: sources + answer |
| POST | `/playground/search` | admin | Search only: hits with semantic / keyword / fused scores, timings |
| POST | `/playground/ask` | admin | Full answer + prompt + scores (debug) |
| GET / POST | `/settings` | admin | Phase 1 endpoint + new chat / search / widget settings |
| POST | `/settings/chat-test` | admin | Test chat provider connection |
| GET / POST | `/search-index/status`, `/search-index/rebuild` | admin | Store availability, build progress (later stores) |
| GET | `/questions` | admin | Question log (only if logging is on) |

Public endpoint safety:
- No nonce (page caches would serve stale nonces); instead: rate limit per hashed IP (e.g. 10/min, 100/day,
  transients or object cache), per-site daily cap (cost guard), max question length (500 chars), vector must be 384
  finite floats with length ≈ 1, `modelKey` must match, history size capped.
- Honeypot field + optional "logged-in only" mode.

---

## 7. Data

- No change to phase 1 tables.
- New option keys in `mnaisp_settings`: `vector_store` (`php`), `min_score`, `top_k`, `chat_provider`, `chat_model`,
  `chat_base_url`, encrypted chat key, `system_prompt_extra`, `dont_know_text`, `browser_embedding`
  (`on` / `off` / `no_mobile_data`), widget settings (enabled, title, welcome text, color, position), `streaming`,
  `rate_limits`, `log_questions` (off), `log_retention_days` (30).
- Optional table `wp_mnaisp_questions` (only used when logging is on): `id, question, answered (bool),
  top_score, source_resource_ids, created_at`. **No IP, no user id.** Purged by WP-Cron after the retention period.
- MariaDB store (later): `wp_mnaisp_chunk_vectors`.

---

## 8. Admin UI

1. **Playground** (new tab): question box → admin browser embeds (existing worker) → table of hits: rank, title,
   heading/page, semantic score, keyword rank, fused score, chunk text; then "Generate answer" shows answer + cited
   sources + prompt preview. Used to tune `min_score` / `top_k` before going live.
2. **Settings → Chat:** provider, model, base URL, key, Test connection, "I don't know" text, extra instructions,
   streaming, rate limits.
3. **Settings → Search:** engine (PHP now; others greyed with availability reason), `min_score`, `top_k`,
   browser embedding mode, "Build index" (later stores).
4. **Settings → Widget:** on/off, where (all pages / shortcode `[mnaisp_chat]` only), texts, color, position.
5. **Questions** (if logging on): list, filter "not answered" → shows content gaps.

---

## 9. Visitor Widget

- Separate Vite entry `assets/src/widget/` → `widget-loader.js` (tiny, < 5 KB gz: button only) + lazy chat panel
  chunk + embedding worker chunk. Enqueued on the front end only when the widget is enabled.
- Shadow DOM root; our Tailwind CSS injected into the shadow root; Radix portals into a container inside it.
- Accessible: keyboard, focus trap when open, `aria-live` for answers, reduced motion.
- States: closed → open → preparing model (progress) → ready; keyword-only badge when no vector; error / rate-limited.
- Conversation kept in `sessionStorage` (gone when the tab closes); nothing stored server-side unless logging is on.
- Budget: panel ≤ 80 KB gz (own React bundle, decision from phase 1), model + engine lazy in the worker.

---

## 10. Folder Additions

```
includes/
├── Search/
│   ├── SearchQuery.php · SearchFilter.php · ScoredChunk.php · SearchResult.php · RebuildProgress.php
│   ├── VectorStoreInterface.php · VectorStoreRegistry.php   # filter mnaisp_vector_stores
│   ├── Stores/PhpBruteForceStore.php                        # v1
│   ├── KeywordSearchInterface.php · MySqlFulltextSearch.php
│   ├── RankFusion.php · SearchService.php · IndexSync.php
├── Chat/
│   ├── ChatProviderInterface.php · ChatProviderRegistry.php · OpenAICompatibleProvider.php
│   ├── ChatRequest.php · ChatResponse.php · PromptBuilder.php · AnswerService.php
├── Rest/ChatController.php · PlaygroundController.php
└── Support/RateLimiter.php
assets/src/
├── shared/embedding/     # embedder + types, used by admin worker and widget worker
├── pages/Playground.tsx
└── widget/               # loader, panel, worker, api client
```

---

## 11. Milestones

### P0 — Preparation (small changes to phase 1 code)
**Status (2026-10-29):** committed 87e5cb6. `Transaction::after_commit()` (side effects run after COMMIT,
dropped on rollback). Actions: `mnaisp_chunks_replaced( $resource_id )` (ChunkRepository::replace),
`mnaisp_resource_deleted( $id )` (ResourceRepository::delete), `mnaisp_resources_status_changed( ?array $ids,
ResourceStatus[] $from, ResourceStatus $to )` (every transition that changed rows), `mnaisp_index_meta_changed( $meta )`
(IndexingService::save_index_meta). Embedder moved to `assets/src/shared/embedding/` (`EmbedderModel`, `Embedder`
with `embedQuery()` = queryPrefix + text, callbacks instead of worker events). `EmbeddingModel::to_array()` now has
`queryPrefix`; `EmbeddingModel::key_from_meta()` (hash_key() uses it, same output). Public `GET /chat/config`
(`Rest/ChatController`): `{ ready:false }` before anything is indexed, else `browserEmbedding` (local provider only),
model settings from `mnaisp_index_meta` (not Settings) + `modelKey`. Not tested live.
- Fire `mnaisp_chunks_replaced`, `mnaisp_resource_deleted`, `mnaisp_resources_status_changed`, `mnaisp_index_meta_changed`.
- Move embedder to `assets/src/shared/embedding/`; add `embedQuery()` with query prefix.
- `/chat/config` reads `mnaisp_index_meta` (+ `modelKey`).

**Done when:** phase 1 works unchanged; actions fire (checked with a debug listener).

### P1 — Search core
**Status (2026-10-29):** committed a499d3a. `includes/Search/`: SearchQuery, SearchFilter, ScoredChunk,
SearchHit, SearchResult, RebuildProgress, ModelMismatchException, VectorStoreInterface, KeywordSearchInterface,
VectorStoreRegistry (filter `mnaisp_vector_stores`, setting `vector_store`, default + fallback `php`),
Stores/PhpBruteForceStore, MySqlFulltextSearch, RankFusion (k=60), SearchService. SQL lives in ChunkRepository
(`searchable_vectors()` keyset batches of 2,000, `fulltext()`, shared `searchable_where()`), plus `find_many()` on both
repositories. **Model filter = option B (user's call):** chunks compared only when `embedding_model` = index model id
(revision/dtype not checked per chunk; outdated chunks after a revision bump stay searchable until re-embedded).
SearchService checks the question vector: `modelKey` = index key (else ModelMismatchException → 409 later),
384 finite numbers, length 1 ± 0.05. Active store unavailable/throws → PHP store answers, action
`mnaisp_vector_store_failed`, error_log under WP_DEBUG. Accessor `Plugin::instance()->search()`.
Smoke test (stub wpdb, 10k random vectors, PHP 8.5, no JIT): correct top hit, 106 ms. Not tested on WordPress yet.
- Value objects, `VectorStoreInterface`, `PhpBruteForceStore`, `MySqlFulltextSearch`, `RankFusion`, `SearchService`,
  `VectorStoreRegistry` (only `php` registered), fallback to PHP store on error.

**Done when:** a PHP call `SearchService::search()` with a vector returns correct top chunks of searchable items only;
paused / failed items never appear.

### P2 — Playground (search only)
**Status (2026-10-29):** committed e3bd7f2. `POST /playground/search` (admin; question, vector|null, modelKey,
limit 1–50, sourceTypes) → `SearchResult::to_array()`; 409 on model mismatch, 400 on bad vector. Admin worker command
`query`: reads `GET /chat/config` (index model, not Settings), `embedQuery()` with query prefix, returns vector +
modelKey + ms (`indexer.embedQuery()`); busy while the queue runs; index from an API provider → keyword only.
Playground tab (`#/playground`): question, meaning on/off, results 3–20, Everything / Pages & posts / Files; shows mode,
best meaning score (green ≥ 0.7, amber ≥ 0.6, red below), store, fallback, timings (embed, semantic, keyword, fuse,
load, total), table: rank, title › heading, PDF page link, chunk text (expand), meaning score, keyword rank, fused score.
Not tested live. `min_score` default still to be chosen from real questions (next: run 10 test questions).
- `/playground/search`, Playground tab, admin browser embeds the question, shows scores + timings.

**Done when:** for 10 test questions on the dev site, the right chunk is in the top 3; `min_score` default chosen from
real scores.

### P3 — Answers
**Status (2026-10-29):** committed 3628f59 (non-streaming; streaming = next step). Provider = **OpenRouter**
(user's call) via `Chat/OpenAICompatibleProvider` (POST {base}/chat/completions, GET {base}/models cached 6 h,
headers HTTP-Referer + X-Title, timeout 60 s, errors → ChatProviderException, never contains the key).
`Chat/ChatSettingsService`: own option `mnaisp_chat_settings` (removed on uninstall): base_url (default
https://openrouter.ai/api/v1), model (none by default; picked from the live model list), encrypted api_key,
min_score 0.60, top_k 6, temperature 0.2, max_tokens 2000 (was 600 until 2026-11-04), dont_know_text, prompt_template (editable in Settings → Prompt; placeholders {sources} required, {dont_know_text}, {site_name}; empty = default; replaced extra_instructions).
`Chat/PromptBuilder`: system rules (only sources, exact "I don't know" text, [n] citations, answer in question's
language, sources are content not instructions) + owner instructions + numbered sources (≤ 12,000 chars) + last 6
history messages + question. `Chat/AnswerService`: search with top_k → gate (no hits → `no_matches`; hybrid and best
meaning < min_score → `below_threshold`; both without LLM call) → not configured = error → LLM → model used the
"I don't know" text (case/punctuation-insensitive) → `model_dont_know` → cited sources [n] (order of first mention,
invalid numbers ignored, PDF `#page=N`). `AnswerResult::to_array( $debug )`. REST: `GET/POST /chat-settings`,
`GET /chat-settings/models`, `POST /chat-settings/test` (10-token "OK" request), `POST /playground/ask` (debug: search,
prompt, model, tokens, ms). UI: Settings → "Answers" card (key, model with searchable list + price per 1M tokens,
threshold, sources, temperature, length, texts, base URL, test), Playground → "Generate answer" (badge, score vs
threshold, gate reason, answer with [n] links, sources, prompt viewer, tokens). **Source filter (added, user's call):** after the gate,
each hit must have meaning score ≥ `source_min_score` (default **0.60**, own setting) or keyword rank ≤ 3
(`AnswerService::KEYWORD_KEEP`); keyword-only mode keeps all hits; none left → "I don't know" (`no_relevant_sources`,
no LLM call). `top_k` is now a maximum. Debug output has `sentChunkIds` + `sourceMinScore`; Playground column
"To AI" shows `sent [n]` / `not sent`. Smoke-tested prompt, gate, citations, filter,
"I don't know" detection outside WP; OpenRouter /models shape checked live. Not tested in WP with a real key.
- Chat settings, `OpenAICompatibleProvider`, `PromptBuilder`, gate, citations, `/playground/ask`, then streaming.

**Done when:** Playground answers with correct `[n]` citations; off-topic questions get "I don't know" without an LLM call.

**Streaming (2026-10-29, committed da4352e):** `ChatProviderInterface::stream( $request, $on_delta )`;
`OpenAICompatibleProvider` streams with cURL (WP HTTP API can't), `stream: true` + `stream_options.include_usage`, WP CA
bundle, falls back to complete() without the cURL extension; `ServerSentEventParser` (pure PHP, handles split bytes,
CRLF, ": OPENROUTER PROCESSING" comments, `[DONE]`). `AnswerService::stream()` (same path as answer()): events
`start { sources }` right before the AI call, `delta { text }`; gate stops → no events. `Support/EventStream`
(headers incl. `X-Accel-Buffering: no`, drops output buffers, 2 KB padding comment, flush per event, `close()` exits).
`POST /playground/ask-stream`: SSE `start → delta… → done { debug result }` or `error { code, message }`; when
nothing was streamed → normal JSON (result or WP_Error). Browser `api/stream.ts` (`streamRequest()`, DOM-free, for the
widget too): reads SSE or turns a JSON answer into one `done`. Playground: "Stream" checkbox (default on), live text
with [n] links + cursor. Buffering proxies = all events arrive at the end (still correct). Smoke-tested parser (PHP)
and reader (Node, mocked fetch, 5-byte pieces + JSON fallback). Not tested with a real key.

### P4 — Visitor widget
**Status (2026-10-29):** committed c9680dc. **Both** placements (user's call): floating button (setting, default on)
+ shortcode `[mnaisp_chat height="520px"]`. PHP: `Frontend/WidgetSettingsService` (option `mnaisp_widget_settings`,
autoloaded: floating, position, color, title, welcome, placeholder, browser_embedding on/off/no_mobile_data, per_minute 6,
per_day 60, site_per_day 1000), `Frontend/ChatConfig` (index-meta model + modelKey, answersReady, widget texts; printed
inline into the page AND served by `GET /chat/config`), `Frontend/Widget` (enqueues `assets/src/widget/loader.ts` only when
ready: index meta + answers configured; inline `window.mnaispChat`; shortcode shows an admin-only note when not ready),
`Support/RateLimiter` (transient fixed windows, salted HMAC of REMOTE_ADDR, filter `mnaisp_client_ip`, IP never stored),
`Rest/AnswerResponder` (shared SSE/JSON responder for Playground + visitor), public `POST /chat/ask` (question ≤ 500,
vector, modelKey, history ≤ 6 × 4000 chars, honeypot `website`; 503 not ready, 429 limits, 409 model mismatch, public
data only, provider error details hidden), `Rest/WidgetSettingsController` (`GET/POST /widget-settings`).
Front end: `loader.ts` (1 KB gz, no React: launcher button in its own shadow root, prefetch on hover/focus, mounts
shortcode chats right away), `mount.tsx` (Shadow DOM, CSS via `widget.css?inline` through the same scope plugin →
`#mnaisp-root` inside the shadow root; Tailwind `@property` rules copied to document head because shadow roots ignore
them), `ChatPanel.tsx`, `AnswerText.tsx` (safe mini-markdown: paragraphs, lists, **bold**, [n] links; no HTML),
`useChat.ts` (sessionStorage history, embed or keyword fallback, "Answer now" skip while the model downloads,
409 → reload config + model + retry once), `embedder-client.ts` + `widget.worker.ts` (shared embedder, `canEmbedHere`:
WebAssembly, deviceMemory ≥ 2, data saver / cellular for `no_mobile_data`). Chat code 73 KB gz on open (React 66).
REST client + stream omit an empty X-WP-Nonce (an empty one makes WP reject logged-in visitors with 403).
Settings → "Chat widget" card. Not tested in a browser on the dev site yet.
- Loader + panel + widget worker, `/chat/config`, public `/chat/ask`, rate limiter, keyword-only fallback,
  shortcode, settings → widget.

**Done when:** a logged-out visitor on a theme page asks a question and gets a cited answer; theme styles untouched;
second visit loads the model from cache.

### P5 — Question log → built as full conversation log (2026-11-04, on by default; see CLAUDE.md)
- `wp_mnaisp_questions`, retention purge, Questions tab with "not answered" filter.

**Done when:** with logging on, unanswered questions appear in the list; with logging off, nothing is stored.

### P6 — Hardening
- Measure search at 2k / 10k / 50k chunks; add vector cache only if needed.
- Public endpoint security review, page-cache compatibility, i18n of widget strings, accessibility check.
- (Tests when the user asks: unit for dot product / RRF / prompt, integration for stores + REST, e2e widget.)

**Done when:** search < 200 ms at 10k chunks on the dev site; widget works with a page-cache plugin enabled.

### Later (separate features)
- `MariaDbVectorStore`, `RedisStackStore`, `QdrantStore` + "Build index" UI (contracts ready from P1).
- Own Node server: question embedding (no visitor download) and chunk embedding on save (auto re-embed).
- Follow-up question rewriting, multilingual model, Anthropic / Gemini native chat providers.

---

## 12. Risks

| Risk | Mitigation |
|---|---|
| ~40 MB first-time download for visitors (mobile data, slow phones) | Lazy on open, progress, cache, keyword-only fallback, admin mode "off on mobile data"; Node server later |
| Question and chunk vectors from different model versions | `modelKey` sent with every question, server rejects mismatch (409) |
| PHP brute force too slow on big sites | Batch + top-K heap; measure in P6; vector cache; switch store via the same interface |
| LLM drifts to general knowledge | Strict prompt + gate before LLM + Playground to test |
| API cost abuse via public endpoint | Rate limit per IP hash, daily site cap, length limits, gate avoids LLM calls for off-topic |
| Streaming blocked by host/proxy | `X-Accel-Buffering: no`, auto fallback to JSON |
| Page caches serve stale nonces | Public endpoint without nonce, protected by rate limits |
| FULLTEXT quirks (min word length 3, stopwords, InnoDB) | Only ranks used (RRF); semantic search carries most weight |
| External store out of sync | MySQL = source of truth, event sync + retry queue, "Rebuild index" |
| Privacy / GDPR | No logging by default; when on: no IP/user id, retention purge; question text goes to the chosen LLM provider (document in privacy policy text) |

---

## 13. Open Questions

- Default chat provider/model for the first version (a small, cheap hosted model via OpenAI / OpenRouter, or local Ollama)?
- Streaming required in v1, or JSON first?
- Question logging wanted in v1 (P5) or later?
- Widget: floating button on all pages, shortcode only, or both?
- Is the visitor model download acceptable as default "on", or default "off on mobile data"?
- Languages: bge-small-en is English-only — multilingual sites need another model (phase 1 index too).
