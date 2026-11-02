# Phase 1 Plan — Content Indexing (RAG Ingestion)

> **Goal:** An admin can choose which pages, posts and files go into the knowledge base, turn them into
> embeddings, store them in the WordPress database, see what is indexed, and re-run indexing on any item.
>
> **Not in phase 1:** visitor chat widget, question embedding, search, answer generation (all phase 2).

Last updated: 2026-10-29

---

## 1. Summary

| Item | Decision |
|---|---|
| Architecture | Layered + modular: REST Controllers → Services → Repositories, React admin app (own React bundle via Vite, not WP's React) |
| Patterns | Strategy (embedding providers), State machine (resource status), Repository (DB), Browser worker (pipeline) |
| Where embedding runs | Admin's browser (Transformers.js in a Web Worker) **or** an API provider via PHP |
| Default model | `Xenova/bge-small-en-v1.5` — 384 dimensions, English, CLS pooling, normalized |
| Storage | Custom WordPress tables (`mnaisp_resources`, `mnaisp_chunks`) |
| Content types | Pages, posts, custom post types, files (PDF, DOCX, TXT, MD) |
| Plugin name | **Compact Local Chat Bot & Knowledgebase** |
| Plugin slug / folder | `compact-local-chatbot` |
| Prefix | `mnaisp` (see naming conventions below) |
| PHP namespace | `Mnaisp\` |
| Text domain | `mnaisp_translation` |
| Requirements | WordPress 6.4+, PHP 8.1+, MySQL 5.7+ / MariaDB 10.3+ |
| Testing | PHPUnit (PHP), Playwright: JS unit in Node (no browser) + e2e in headless Chromium |

---

## 1a. Naming Conventions

All code lives in namespaced classes. The prefix is only used where WordPress shares a global space.

| Thing | Rule | Example |
|---|---|---|
| PHP classes | Namespace `Mnaisp\`, no prefix in class name | `Mnaisp\Services\IndexingService` |
| Methods | No prefix | `IndexingService::claim_next()` (snake_case, per WordPress Coding Standards) |
| Global functions | Avoid; if unavoidable, prefix | `mnaisp_activate()` |
| Constants | `MNAISP_` | `MNAISP_VERSION`, `MNAISP_PLUGIN_DIR` |
| DB tables | `{$wpdb->prefix}mnaisp_` | `wp_mnaisp_resources` |
| Options / transients | `mnaisp_` | `mnaisp_settings`, `mnaisp_index_meta` |
| Hooks (actions/filters) | `mnaisp_` | `mnaisp_before_index`, `mnaisp_chunk_text` |
| REST namespace | `mnaisp/v1` | `/wp-json/mnaisp/v1/resources` |
| Script / style handles | `mnaisp-` | `mnaisp-admin` |
| JS globals (localized data) | `mnaispData` | `window.mnaispData.restUrl` |
| CSS classes | `mnaisp-` | `.mnaisp-status-badge` |
| Text domain | `mnaisp_translation` | `__( 'Indexed', 'mnaisp_translation' )` |

> **Note:** WordPress.org requires the text domain to equal the plugin slug (`compact-local-chatbot`).
> `mnaisp_translation` is fine for private / client distribution; change it before submitting to WordPress.org.

---

## 2. Architecture

```
┌─ Admin dashboard (React app) ────────────────────────────────┐
│  Overview · Add Content · Knowledge Base · Settings           │
│                                                               │
│  Web Worker (embedding pipeline)                              │
│   get text → chunk → embed → save in batches                  │
│        │                       │                              │
│   Local: Transformers.js       API: POST /embed (PHP calls    │
│   bge-small-en-v1.5            provider, key stays on server) │
└────────┼───────────────────────┼──────────────────────────────┘
         │      WordPress REST API  (namespace mnaisp/v1)
┌────────▼──────────────────────────────────────────────────────┐
│  Rest/          Controllers: validate, check capability        │
│  Services/      Resources, Indexing, Settings, ContentExtractor│
│  Providers/     EmbeddingProviderInterface + implementations   │
│  Repositories/  ResourceRepository, ChunkRepository            │
│  Hooks/         ContentWatcher (save/delete → outdated/remove) │
│  Database       mnaisp_resources · mnaisp_chunks                     │
└───────────────────────────────────────────────────────────────┘
```

### Resource status (state machine)

```
not indexed ──add──▶ queued ──claim──▶ processing ──success──▶ indexed
                       ▲                    │                      │
                       │                    └──error──▶ failed     │
                       │                                  │        │
                       └────────── re-embed ──────────────┴────────┤
                                                                   ▼
                              content edited / model changed ─▶ outdated
```

- `processing` items not finished within 10 minutes go back to `queued` (tab closed, browser crash).
- `outdated` keeps the old chunks until new ones are saved, so nothing is lost.
- `paused` (added 2026-10-29): source unpublished (draft, pending, private, trash), from any status. Chunks kept, never
  searched (phase 2 searches only `ResourceStatus::searchable()` = indexed + outdated). Publish again →
  `indexed` (text + title unchanged), `outdated` (changed; unchanged chunks reuse vectors) or `queued` (no chunks yet).
  Model change during pause clears its `content_hash`, so it resumes as `outdated`. Not re-embeddable while paused.
  Permanent delete → removed.

---

## 3. Folder Structure

```
compact-local-chatbot/
├── compact-local-chatbot.php              # plugin header, bootstrap
├── uninstall.php               # drop tables + options (if user opts in)
├── composer.json               # PSR-4 autoload: Mnaisp\ → includes/
├── package.json                # Vite build, own React bundle (react + react-dom)
├── vite.config.ts
├── includes/
│   ├── Plugin.php
│   ├── Admin/AdminPage.php     # menu + mounts React app
│   ├── Rest/
│   │   ├── SettingsController.php
│   │   ├── SourcesController.php     # WP content available to add
│   │   ├── ResourcesController.php   # knowledge base items
│   │   ├── JobsController.php        # claim / save / fail
│   │   └── EmbedController.php       # API provider proxy
│   ├── Services/
│   │   ├── ResourceService.php
│   │   ├── IndexingService.php
│   │   ├── SettingsService.php
│   │   └── ContentExtractor.php      # post → clean text
│   ├── Providers/
│   │   ├── EmbeddingProviderInterface.php
│   │   ├── ProviderRegistry.php
│   │   ├── LocalBrowserProvider.php  # metadata only, runs in browser
│   │   └── OpenAIProvider.php
│   ├── Repositories/
│   │   ├── ResourceRepository.php
│   │   └── ChunkRepository.php
│   ├── Database/Migrations.php
│   ├── Domain/ResourceStatus.php     # PHP enum
│   ├── Hooks/ContentWatcher.php
│   └── Support/
│       ├── Crypto.php                # API key encryption
│       └── VectorCodec.php           # float32 ⇄ binary
├── assets/src/
│   ├── index.js
│   ├── api/client.js
│   ├── pages/  Overview, AddContent, KnowledgeBase, Settings
│   ├── components/  StatusBadge, ProgressPanel, ChunkViewer, ResourceTable
│   │   └── ui/      # shadcn/ui components (Radix + Tailwind), owned by us
│   └── worker/
│       ├── embed.worker.js           # Web Worker entry
│       ├── pipeline.js
│       ├── chunker.js
│       ├── extractors/  pdf.js, docx.js, text.js
│       └── providers/   local.js, api.js
├── tests/
│   ├── phpunit/
│   │   ├── bootstrap-unit.php · bootstrap-integration.php · wp-tests-config.php
│   │   ├── Unit/          # pure PHP: VectorCodec, ResourceStatus, Crypto
│   │   └── Integration/   # with WordPress: repositories, services, REST, hooks
│   ├── js-unit/           # Playwright runner, NO browser: chunker, extractors, codec
│   └── e2e/               # Playwright, headless Chromium: specs + fixtures (sample PDF, DOCX, TXT)
├── phpunit.xml.dist             # unit (no WP)
├── phpunit-integration.xml.dist # integration (WP test suite via wp-phpunit)
├── playwright.config.ts
├── postcss.config.js · tools/postcss-scope-to-root.js   # Tailwind scoped to #mnaisp-root
└── tools/                 # build helpers
```

---

## 4. Database

### `wp_mnaisp_resources` — one row per page, post or file

| Column | Type | Notes |
|---|---|---|
| id | BIGINT PK | |
| source_type | VARCHAR(10) | `post` (any post type) or `file` |
| source_id | BIGINT | post ID or attachment ID |
| post_type | VARCHAR(20) NULL | `post`, `page`, `<cpt>`; NULL for files |
| title | VARCHAR(255) | |
| mime_type | VARCHAR(100) NULL | files only |
| status | VARCHAR(20) | see state machine |
| content_hash | CHAR(64) NULL | SHA-256 of extracted text, detects real changes |
| chunk_count | INT | |
| embedding_model | VARCHAR(100) NULL | model used for current chunks |
| embedding_dims | SMALLINT NULL | |
| error_message | TEXT NULL | |
| attempts | TINYINT | failed / abandoned tries. Error → `failed` at once; abandoned claim → back to `queued`, `failed` after 3. Re-embed resets to 0 |
| claimed_at | DATETIME NULL | lock for `processing` |
| indexed_at | DATETIME NULL | |
| created_at / updated_at | DATETIME | |

Unique key: `(source_type, source_id)`. Indexes: `status`, `post_type`.

No URL column (removed in schema v2, 2026-10-29): the link is built at runtime from `source_id`
(`get_permalink()` / `wp_get_attachment_url()`, `SourceService::url_for()`), so permalink, slug or domain changes never
leave stale citation links.

### `wp_mnaisp_chunks` — pieces of text with their embeddings

| Column | Type | Notes |
|---|---|---|
| id | BIGINT PK | |
| resource_id | BIGINT | FK → resources (cascade delete in code) |
| chunk_index | INT | order inside resource |
| content | MEDIUMTEXT | chunk text shown in citations |
| heading | VARCHAR(255) NULL | nearest heading |
| page_number | INT NULL | PDFs, for `file.pdf#page=N` links |
| token_count | INT | |
| chunk_hash | CHAR(64) | `sha256(model_key + "\|" + model_input)`, model_key = `EmbeddingModel::hash_key()` = id:revision:dtype:pooling:normalize (sent to the worker as `model.hashKey`), model_input = title › heading › chunk (same for local + API; never stored). Same hash → reuse old vector. Key is only in the hash, never sent to the model. (Before 2026-10-29 only the model id was in the hash, so a new revision/dtype reused stale vectors — fixed.) |
| embedding | BLOB | packed float32 (384 × 4 = 1,536 bytes); numbers only, no text |
| embedding_model | VARCHAR(100) | |
| created_at | DATETIME | |

Indexes: `resource_id`, `(resource_id, chunk_hash)`; `FULLTEXT(content)` — ready for phase 2 keyword search.

Relation: `chunks.resource_id` → `resources.id` (one resource, many chunks). No DB foreign key (`dbDelta` /
MyISAM hosts); `ChunkRepository` deletes chunks with their resource.

### Option `mnaisp_index_meta`
Model id, dims, dtype, pooling, normalize, query prefix. Phase 2 question embedding must match this exactly.

---

## 5. Settings Panel

| Setting | Values | Notes |
|---|---|---|
| Provider | Local (browser), OpenAI | More providers later via Strategy pattern |
| Model | Local: `bge-small-en-v1.5`; OpenAI: `text-embedding-3-small` | List depends on provider |
| API key | text | Encrypted at rest (libsodium, key from WP salts); UI shows only `sk-…abcd` |
| Test connection | button | Embeds "hello" and checks dimensions |
| Content types allowed | post types + file types | |
| Chunk size / overlap | default ~250 words / ~40 words | Advanced section |
| Max file size | default 20 MB | |

**Changing provider or model** → warning dialog → all `indexed` resources become `outdated`
(vectors from different models cannot be mixed).

---

## 6. REST API (`/wp-json/mnaisp/v1`)

All endpoints require `manage_options` + REST nonce.

| Method | Route | Purpose |
|---|---|---|
| GET / POST | `/settings` | Read / save settings |
| POST | `/settings/test` | Test provider connection |
| GET | `/sources?type=&search=&page=` | WP content + files with their index status |
| GET | `/resources?status=&type=&page=` | Knowledge base list |
| POST | `/resources` | Add selected items → `queued` |
| DELETE | `/resources/{id}` | Remove from KB (deletes chunks) |
| POST | `/resources/{id}/reindex` | → `queued` |
| POST | `/resources/reindex` | Bulk: by ids or all `outdated` / `failed` |
| GET | `/resources/{id}/chunks` | Chunk viewer |
| GET | `/stats` | Counts per status, total chunks, model |
| POST | `/jobs/claim` | Worker claims next N `queued` items → `processing` |
| GET | `/jobs/{id}/content` | Clean text (posts) or file URL + mime (files) |
| POST | `/jobs/{id}/complete` | Save chunks + vectors, replace old ones in one transaction → `indexed` |
| POST | `/jobs/{id}/fail` | Save error → `failed` |
| POST | `/embed` | API provider only: texts in, vectors out |

Vectors travel as base64-encoded Float32Array.

---

## 7. Embedding Pipeline

Runs in a **Web Worker** so the dashboard never freezes.

1. **Claim** a small batch of queued resources (e.g. 3).
2. **Get text**
   - Posts / pages: PHP renders blocks + shortcodes (`the_content`), converts headings to markers, strips HTML, normalizes whitespace.
   - PDF: `pdfjs-dist`, text per page, keep page numbers.
   - DOCX: `mammoth` → HTML → text with headings.
   - TXT / MD: read directly.
   - No text found (scanned PDF) → `failed`: "No text found — scanned PDFs are not supported yet."
3. **Chunk**
   - Split on headings, then paragraphs, then sentences.
   - Target ~250 words, ~40 words overlap.
   - Hard cap 512 tokens (checked with the model tokenizer for local provider).
   - Embedded text = `title + heading + chunk` (better matches); stored text = chunk only.
4. **Embed**
   - Local: `pipeline("feature-extraction", "Xenova/bge-small-en-v1.5", { dtype: "q8" })`,
     `{ pooling: "cls", normalize: true }`, batches of 16.
   - API: POST `/embed` in batches; retry with backoff on 429 / 5xx.
5. **Save** all chunks of a resource in one request → old chunks replaced atomically.
6. Repeat until queue empty.

**First run:** model (~35 MB) downloads from Hugging Face and is cached by the browser; show download progress.
**UX:** progress panel, pause / resume, warning before leaving page while running.

---

## 8. Admin UI Screens

1. **Overview**: counts (indexed, queued, outdated, failed), total chunks, current model, big "Process queue" button, progress panel.
2. **Add Content**: tabs Pages · Posts · Custom types · Files. Search, filter, checkboxes, status badge per row, "Add to knowledge base".
3. **Knowledge Base**: table with title, type, status, chunks, model, last indexed. Row actions: View chunks · Re-embed · Remove. Bulk: re-embed selected, re-embed all outdated, retry failed.
4. **Chunk Viewer** (side drawer): every chunk with heading, page number, word count.
5. **Settings**: see section 5.

### Frontend stack

- **Own React bundle:** `react` + `react-dom` from npm, bundled with **Vite** into `assets/build/`.
- **Not used:** WordPress's built-in React (`wp-element`), `@wordpress/scripts`, `@wordpress/components`,
  `@wordpress/api-fetch`. Our bundle does not depend on, or register, any `wp-*` script.
- **Why:** we control the React version and upgrades; no breakage when WordPress core updates its React.
- **Consequences:**
  - REST calls use plain `fetch` with the `X-WP-Nonce` header (nonce + REST URL passed via `window.mnaispData`).
  - App mounts into one root element: `<div id="mnaisp-root">` on our admin page only.
  - Script enqueued only on our admin screens (handle `mnaisp-admin`), never site-wide.
  - Bundle size budget: React + UI ≈ 80–120 KB gzipped (Transformers.js and pdf.js load lazily in the worker).

### UI components: shadcn/ui (built on Radix UI)

- **shadcn/ui** = ready-made components copied into our code (`assets/src/components/ui/`), built on
  **Radix UI** primitives (accessibility, keyboard, focus handling) and styled with **Tailwind CSS**.
- We own the component code: change freely, no locked library version.
- Components we need: Button, Input, Select, Checkbox, Table, Tabs, Dialog, Sheet (side drawer for Chunk Viewer),
  Badge (status), Progress, Alert, Tooltip, Toast, Dropdown Menu, Skeleton (loading).
- Tables with sort / filter / pagination: shadcn Data Table pattern (TanStack Table).

**Living inside WP admin — rules:**
- **Tailwind reset (preflight) scoped** to `#mnaisp-root` only. A global reset would break the WP admin menu and other plugins.
- **Tailwind utilities scoped** to `#mnaisp-root` (or given a `mnaisp-` prefix); exact Tailwind config fixed in M0 spike.
- **Radix portals** (Dialog, Sheet, Dropdown, Tooltip, Toast) render into `document.body` by default — outside our
  scope, so they'd lose our styles. Set every portal `container` to our root element.
- Raise z-index above the WP admin bar (`#wpadminbar` ≈ 99999) for dialogs and toasts.
- Theme tokens (CSS variables) set on `#mnaisp-root`, matching WP admin colors where sensible.

---

## 9. Keeping Content in Sync (hooks)

| WordPress event | Action |
|---|---|
| `save_post` on indexed item | Re-extract text, compare hash → `outdated` only if text really changed |
| Unpublish / trash (`transition_post_status`) | → `paused`, chunks kept |
| Publish again (`save_post`) | `paused` → `indexed` / `outdated` / `queued` |
| `before_delete_post` (permanent delete) | Remove resource + chunks |
| `delete_attachment` | Remove resource + chunks |
| Attachment file replaced | → `outdated` |
| Embedding model/provider changed | All → `outdated` |

Phase 1 marks items as outdated; the admin re-runs them (banner: "5 items need re-embedding").
Automatic re-embedding comes once the server-side embedding service exists.

---

## 10. Milestones

### M0 — Project setup
- Plugin skeleton, Composer PSR-4 autoload
- Vite build with own `react` + `react-dom` bundle (no `wp-element`); dev mode with hot reload
- Tailwind + shadcn/ui setup; spike: scoped preflight, portals into `#mnaisp-root`, one Dialog renders correctly in WP admin
- PHPCS (WordPress Coding Standards), ESLint
- Tests run against the existing dev site (docker compose, `~/Desktop/dev-site`); PHPUnit + Playwright wired up, smoke tests each
- Activation / deactivation / uninstall handlers

**Done when:** plugin activates cleanly, empty React admin page renders under the "Compact Chat Bot" menu.

**Status (2026-10-28):** code done in `~/Desktop/dev-site/wp/wp-content/plugins/compact-local-chatbot/`.
Verified: PHPUnit unit 4/4, Playwright unit 8/8, PHPCS, ESLint, tsc, build (93 KB gzip), built CSS 100% scoped.
Pending: activation in WP + PHPUnit integration (manual local testing by user).
E2E deferred (2026-10-28): user tests the admin app manually for now; Playwright e2e project + specs added later.

### M1 — Database & core
**Status (2026-10-28):** code done (Migrations, Tables, Transaction, ResourceStatus, Resource, Chunk, VectorCodec,
ResourceRepository, ChunkRepository; stale recovery runs at the start of every claim). Unit tests for status, codec,
chunk hash pass. Integration tests for repositories postponed (user: "will write test later").
- Tables via `dbDelta` with versioned migrations
- `ResourceRepository`, `ChunkRepository`, `VectorCodec`
- `ResourceStatus` enum + allowed transitions
- Stale-claim recovery (processing > 10 min → queued)

**Done when:** unit tests cover repositories, encoding and every status transition.

### M2 — Settings
**Status (2026-10-28):** committed 7fe69b1. SettingsService (one option, API keys encrypted per provider),
Crypto (libsodium secretbox, key from `wp_salt('secure_auth')`), ProviderRegistry + Local/OpenAI metadata
(filter `mnaisp_embedding_providers`), REST `GET/POST /settings`, React Settings page (hash route `#/settings`),
model-change confirm dialog → `mark_all_outdated()`. Bundle 128 KB gzip (over 120 KB budget → lazy-load pages later).
- `SettingsService`, `Crypto`, `ProviderRegistry` + interface
- Settings REST endpoints + Settings screen
- Test connection, model-change warning → mark outdated
- **NOTE:** "Test connection" button is in the UI but **disabled** (markup only, tooltip "Available soon").
  Wire it up in M4 (local model: embed "hello" in the worker, check 384 dims) and M6 (OpenAI via `/settings/test`).

**Done when:** admin saves provider/model/key; key is encrypted in DB and masked in UI.

### M3 — Content selection
**Status (2026-10-28):** committed ca43d78. `GET /sources/types` (one tab per allowed post type + Files),
`GET /sources` (published posts / attachments of allowed MIME types, with KB status, one batch lookup per page),
`POST /resources` (checks type, published, file size; returns added / existing / skipped with reason).
Add Content page (`#/add`): tabs, debounced search, selection kept across tabs + pages, status badges.
Only **published** posts can be added (drafts/private excluded). Pages are lazy-loaded chunks (main 116 KB gzip).
- `/sources` endpoint (posts, pages, CPTs, attachments filtered by allowed types)
- Add Content screen with search, tabs, bulk add

**Done when:** admin selects 10 pages + 2 PDFs and they appear as `queued`.

### M4 — Pipeline for posts & pages (local model)
**Status (2026-10-28):** committed ebb0c79. ContentExtractor (the_content → text, `## Heading` markers,
NFC + whitespace normalized), IndexingService + `/jobs/{config,claim,{id}/content,{id}/complete,{id}/fail}` + `/stats`.
Complete locks the row (`SELECT … FOR UPDATE`) → 409 if removed / re-queued meanwhile, nothing saved.
Unchanged chunks (same hash) reuse stored vectors. `mnaisp_index_meta` written on complete.
Worker (`assets/src/worker/`): DOM-free chunker / encoding / pipeline + Transformers.js embedder; claims 1 post at a time;
store `lib/indexer.ts` keeps progress across tabs; beforeunload warning. "Test connection" works for Local (done).
ONNX Runtime engine (.wasm + .mjs) loads from jsDelivr CDN at the exact version pinned by
Transformers.js (user choice, option B); Vite plugin in vite.config.ts stops the 27 MB wasm being copied into the build.
Known limits: page builders that don't render through `the_content` (e.g. Elementor) may give little text.
- ~~**TODO from M2:** enable "Test connection" for the Local provider~~ done.
- `ContentExtractor` (blocks, shortcodes, page builders)
- Web Worker, chunker, local provider (Transformers.js)
- Claim / complete / fail endpoints, atomic chunk replace
- Progress panel, pause / resume, model download progress

**Done when:** 50 pages index end-to-end in the browser; chunks + vectors visible in DB; closing tab mid-run recovers.

### M5 — Files
**Status (2026-10-29):** committed 74e7156. Claim now hands out posts **and** files. `/jobs/{id}/content`
returns `kind: "text"` (posts, text from PHP) or `kind: "file"` (media URL, MIME, max bytes; type + size re-checked
against Settings, else `failed`). Worker downloads the file (same-origin fetch, size limit) and extracts text in
`assets/src/worker/extractors/`: PDF = `pdfjs-dist` 6.4.299 legacy build, run on the worker's own thread (no nested
worker), lines/paragraphs rebuilt from layout, big-font short lines → `## Heading`, page-number lines dropped,
hyphenation undone; DOCX = `mammoth` 1.13.0 → HTML → `text-normalize.ts` (TS port of PHP `html_to_text`/`normalize`),
images skipped; TXT (UTF-8, fallback Windows-1252); MD (headings kept, links/emphasis/code marks stripped, front matter
dropped). Page numbers: extractor writes a page-marker paragraph (U+0001 + N) before each page; chunker sets
`pageNumber` = page where the chunk's own text starts. Content hash for files = SHA-256 of extracted text (worker).
Scanned PDF (< 5 words per page) → failed "No text found — this looks like a scanned PDF…". Password PDF / invalid PDF /
old .doc → clear error. pdf.js cmaps + standard fonts load from jsDelivr at the exact version, only when a PDF needs
them. Overview shows stage ("Downloading file…", "Reading page 3 of 40…"). pdf.js (~1.7 MB) + mammoth (~310 KB) are
lazy chunks in the worker only; main bundle unchanged (98 KB gzip).
Smoke-checked in Node with generated PDF / scanned PDF / DOCX / MD / TXT (no test files written, tests postponed).
Known limits: media on another domain (CDN / offload plugin) needs CORS; multi-column PDFs follow the PDF's text order;
PDF tables become plain lines; DOCX table cells become separate paragraphs (same as posts).
- PDF (`pdfjs-dist`), DOCX (`mammoth`), TXT/MD extractors
- Page numbers on PDF chunks, scanned-PDF detection, size limit

**Done when:** sample PDF, DOCX, TXT index correctly; scanned PDF fails with clear message.

### M6 — API provider
**Status (2026-10-29):** SKIPPED for now (user's call); come back later. "Test connection" for OpenAI stays disabled.
- **TODO from M2:** enable "Test connection" for OpenAI (`POST /settings/test`, PHP calls provider, checks dimensions).
- `OpenAIProvider` + `/embed` proxy, batching, retry/backoff
- Same pipeline, provider switched by setting

**Done when:** same content indexes via OpenAI; key never appears in browser network traffic.

### M7 — Knowledge base management
**Status (2026-10-29):** committed a459fc7. REST: `GET /resources` (status, type = post type or `file`, search,
page), `DELETE /resources/{id}`, bulk remove `POST /resources/delete` (POST, not DELETE: proxies may drop DELETE bodies),
`POST /resources/{id}/reindex` (409 if queued/processing), `POST /resources/reindex` (ids or status
outdated/failed/indexed), `GET /resources/{id}/chunks`. Knowledge Base screen (`#/knowledge-base`): status + type
filters, search, row actions (view chunks, re-embed/retry, edit in WP, remove), bulk re-embed/remove, "Retry all failed",
confirm dialog for remove. Chunk Viewer = Sheet drawer (heading, PDF page link `#page=N`, words, tokens).
Re-embed queues and starts the worker right away (Local provider). Outdated banner on Overview + KB.
`Hooks/ContentWatcher` (runs on every request, hooked on init): save_post → title refreshed; indexed post → outdated
if text hash or title changed; publish → anything else (draft, pending, private, trash) → **paused** (chunks kept,
not searched; republish resumes, see state machine); before_delete_post / delete_attachment → removed; attachment title change or new
`_wp_attached_file` → outdated. Limit: a file replaced in place under the same name (no meta change) isn't detected.
Not tested live yet (no docker access from Claude's shell).
- **TODO (user, 2026-10-28):** remove from knowledge base — works for `queued` items (no chunks yet) and items with chunks.
  `DELETE /resources/{id}` + bulk delete; confirm dialog. Removing a `processing` item: `/jobs/{id}/complete` must
  lock the row and discard the result if the resource is gone (guard built in M4).
- Knowledge Base screen, Chunk Viewer, re-embed (single + bulk), remove
- `ContentWatcher` hooks, outdated banner, Overview stats

**Done when:** editing an indexed page marks it outdated; re-embed refreshes it; deleting a page removes its chunks.

### M8 — Hardening & testing
**Status (2026-10-29):** security pass DEFERRED (user: "will be done later"). Tests still postponed.
- Security pass: capabilities, nonces, input sanitizing, output escaping
- Load test: 500 pages + 20 PDFs, measure time, DB size, browser memory
- Fill test gaps per section 10a; full Playwright suite green; manual QA checklist
- Translation-ready strings (i18n), uninstall cleanup verified

**Done when:** QA checklist passes on a fresh WordPress install and on one real client site copy.

---

## 10a. Testing Strategy

Two tools only: **PHPUnit** for PHP, **Playwright** for JavaScript.
Playwright is used two ways:
- **JS unit tests** (`tests/js-unit/`): code that does not touch the DOM runs directly in Node — **no browser**. Fast.
- **E2E tests** (`tests/e2e/`): full flows in a real **headless browser** against WordPress.

Rule: if a module or function doesn't use the DOM, test it without the browser. Only UI flows go to e2e.

PHPUnit integration tests and e2e tests run against the existing dev site (docker compose in `~/Desktop/dev-site`).
Integration tests share the dev database but use table prefix `wptests_` (dev `wp_` tables untouched).

### PHPUnit (`tests/phpunit/`)

| Layer | What we test | Examples |
|---|---|---|
| Unit (no WordPress) | Pure logic | `VectorCodec` round-trip float32 ⇄ binary; `ResourceStatus` allowed / forbidden transitions; `Crypto` encrypt ⇄ decrypt |
| Integration (WP test suite) | Code that touches WordPress / DB | Migrations create tables; repositories CRUD; atomic chunk replace; stale-claim recovery |
| REST | Every endpoint | 403 for non-admins; input validation; `/jobs/claim` never gives one item to two workers |
| Hooks | `ContentWatcher` | Edit text → `outdated`; edit without text change → stays `indexed`; delete post → chunks gone |
| Settings | `SettingsService` | Key stored encrypted, returned masked; model change marks all `outdated` |
| Extractor | `ContentExtractor` | Blocks + shortcodes rendered; headings kept; HTML stripped |

Run: `composer test` (unit) · integration inside the WP container: `docker compose exec -w /var/www/html/wp-content/plugins/compact-local-chatbot wordpress php vendor/bin/phpunit -c phpunit-integration.xml.dist`.

### Playwright — JS unit, no browser (`tests/js-unit/`)

| Spec | What we test |
|---|---|
| `chunker.spec.ts` | Chunk sizes, overlap, 512-token cap, heading carry-over, empty / huge input |
| `text-normalize.spec.ts` | Whitespace cleanup, heading markers, special characters |
| `extract-pdf.spec.ts` | Text per page + page numbers from fixture PDF; scanned PDF → "no text" |
| `extract-docx.spec.ts` | Headings + paragraphs from fixture DOCX |
| `vector-codec.spec.ts` | Float32Array ⇄ base64 round-trip, matches PHP `VectorCodec` byte format |
| `pipeline.spec.ts` | Pipeline order + retries with mocked API client and mocked embedder |
| `embedder.spec.ts` | bge-small output: 384 dims, normalized (length ≈ 1), same text → same vector (`@slow`) |

Keep these modules DOM-free on purpose (no `window`, no `document`) so they stay testable in Node.

Run: `npm run test:unit` (→ `playwright test --project=unit`).

### Playwright — E2E, headless browser (`tests/e2e/`) — DEFERRED, manual testing for now

| Spec | Scenario |
|---|---|
| `settings.spec.ts` | Save provider/model/key; key shown masked; test connection; model change warning |
| `add-content.spec.ts` | Search + select pages/posts/files → appear as `queued` |
| `index-posts.spec.ts` | Process queue with local model → items `indexed`, chunk counts shown |
| `index-files.spec.ts` | Sample PDF, DOCX, TXT index; scanned PDF fails with clear message |
| `knowledge-base.spec.ts` | Chunk viewer opens; re-embed single + bulk; remove item |
| `sync.spec.ts` | Edit indexed page in editor → `outdated` banner; delete page → gone from list |
| `recovery.spec.ts` | Close tab mid-run → reopen → stale items back in queue, run completes |
| `worker.spec.ts` | Web Worker runs in the page: chunk + embed a small post end-to-end, UI stays responsive |
| `api-provider.spec.ts` | OpenAI provider with mocked endpoint (`page.route`); key never in browser requests |

Browser setup:
- `playwright.config.ts` has two projects: `unit` (testDir `tests/js-unit`, no browser) and `e2e` (testDir `tests/e2e`, Chromium).
- All e2e tests run in a real browser, **headless** by default (`headless: true`).
- Browser: Chromium. Transformers.js runs on WebAssembly (WASM), which works in headless Chromium; no GPU needed.
- Debug locally with `npx playwright test --headed` or `--ui`; on failure keep trace, screenshot and video.
- Login once as admin in a global setup step and reuse the saved session (`storageState`) for all specs.

Notes:
- Fixtures: small sample PDF, DOCX, TXT, a scanned PDF, and seeded pages/posts.
- Model download is slow: cache the model files between runs; tag the full local-model specs `@slow`.
- API provider tests use a mocked server, never a real paid API.
- Run: `npm run test:e2e` (→ `playwright test --project=e2e`). All JS: `npm run test:js`.

### Per milestone

Each milestone's "done when" must be covered by at least one automated test (PHPUnit, Playwright unit or Playwright e2e) before moving on.

---

## 11. Phase 1 Definition of Done

- [ ] Admin selects pages, posts, CPTs and files from a clear UI
- [ ] Content is chunked and embedded locally in the browser (bge-small-en-v1.5) or via API
- [ ] Chunks + vectors + model info stored in custom tables
- [ ] Admin sees every indexed item, its status, chunk count and chunks
- [ ] Re-embed works per item and in bulk; remove works
- [ ] Edited / deleted content is detected
- [ ] Settings for provider, model and API key work; key is encrypted
- [ ] Index metadata saved so phase 2 can embed questions identically
- [ ] PHPUnit, Playwright unit and Playwright e2e suites pass on a clean WordPress install
- [ ] All strings translatable with text domain `mnaisp_translation`

---

## 12. Risks

| Risk | Mitigation |
|---|---|
| Admin closes tab mid-run | Batches saved per resource; stale claims return to queue |
| Big PDFs use too much browser memory | Process page by page; size limit; warn |
| Model download blocked (firewall, CSP) | Clear error; later option to host model files on own server |
| Tailwind / shadcn styles clash with WP admin or other plugins | Scoped preflight + utilities, portals into our root; e2e screenshot check of WP menu |
| Page builders render content oddly | Test Elementor, Divi, Gutenberg; fallback to stored post content |
| Model mismatch in phase 2 | `mnaisp_index_meta` + model column on every chunk |
| Large index slows phase 2 search | Measure in M8; plan vector search strategy in phase 2 |

---

## 13. Open Questions

- Final plugin name and slug
- Which API providers besides OpenAI for phase 1 (Gemini, Voyage, Mistral)?
- Do we need private/draft content, or published only?
- Multisite support needed?
- Phase 2: question embedding in visitor's browser vs own Node server

---

## 14. Future TODOs

### Self-host the embedding model (no dependency on Hugging Face)
Added 2026-10-28. Today Transformers.js downloads the model from `https://huggingface.co/Xenova/bge-small-en-v1.5/resolve/main/…`
the first time each browser runs it. If Hugging Face is down, blocked, or the repo changes, indexing breaks or vectors differ.

Plan:
- Host the model files on **our own server** (plugin site / CDN we control), at a fixed version:
  `config.json`, `tokenizer.json`, `tokenizer_config.json`, `onnx/model_quantized.onnx` (q8, ~34 MB), keeping the same folder layout.
- Point Transformers.js at it: `env.remoteHost = 'https://<our-host>/'` + `env.remotePathTemplate = '{model}/{revision}/'`
  (or `env.localModelPath` + `env.allowLocalModels = true` when served from the WP site itself).
- Copy files from the **pinned commit** `ea104dacec62c0de699686887e3f920caeb4f3e3` (already pinned in code since 2026-10-28:
  `LocalBrowserProvider` → `revision`, passed to `pipeline()`, saved in `mnaisp_index_meta`).
- Store the host + revision in `mnaisp_index_meta`, so phase 2 (question embedding) loads exactly the same files.
- Optional: keep Hugging Face as a fallback when our host fails.
- Same idea later for the ONNX Runtime engine (`.wasm` + `.mjs`, now from jsDelivr) if we want zero third-party requests.
- Changing host or revision = model change → all indexed items marked outdated → re-embed.
