# How Content Becomes Chunks and Vectors — Chunking Process

How a page, post or file is turned into small pieces of text ("chunks"), and how each piece gets its vector, step by
step. Written for later study: every **bold term** is explained in the glossary at the end.

Code: `includes/Services/ContentExtractor.php` (posts → text, PHP), `assets/src/worker/extractors/` (files → text),
`assets/src/worker/text-normalize.ts`, `assets/src/worker/chunker.ts` (the chunker), `assets/src/worker/pipeline.ts`
(the indexing loop), `assets/src/worker/encoding.ts` + `includes/Support/VectorCodec.php` (vector bytes).
Related: `docs/search-flow.md` (how chunks are searched), `docs/source-selection.md` (how chunks reach the AI),
`docs.html` (the tables).

Last updated: 2026-11-04 (chunker text format 3)

---

## 1. The big picture

```
WordPress post / page                PDF / DOCX / TXT / MD file
   │ PHP ContentExtractor               │ browser downloads it, extractors/ read it
   │ (the_content, tables → rows)       │ (PDF: page markers)
   └───────────────┬────────────────────┘
                   ▼
   clean text: paragraphs, "## Heading", "- list item", "Label: value; …" rows
                   │ chunker.ts (browser Web Worker)
                   ▼
   chunks of ~250 words, 40 words overlap, each ≤ 512 tokens as "Title › Heading › text"
                   │ chunk_hash: unchanged chunks keep their old vector
                   ▼
   embedding model  ──▶  384 numbers per chunk (float32) = 1,536 bytes
   (local: Transformers.js in the browser · API: OpenRouter through PHP)
                   │
                   ▼
   wp_mnaisp_chunks: content (text) + embedding (BLOB) + heading, page, tokens, hash
```

Two things are easy to mix up:

| | Size | Decided by |
|---|---|---|
| **Chunk text** (`content` column) | ~250 words, about 1,000–1,800 characters; small chunks much less | The chunker (words + tokens) |
| **Vector** (`embedding` column) | Always 384 × 4 = **1,536 bytes** (bge-small) | The model only |

The chunker never counts bytes. A 5-word chunk and a 300-word chunk both get a 1,536-byte vector. The two numbers
being close is a coincidence. Another model changes the vector size (text-embedding-3-small: 1,536 numbers =
6,144 bytes), not the chunk size.

---

## 2. Step 1 — Clean text from the source

Both paths produce the **same text format**, so posts and files are chunked by the same rules:

| Format rule | Example |
|---|---|
| Paragraphs separated by one blank line | `First paragraph.\n\nSecond paragraph.` |
| Headings as their own paragraph, one `#` per level | `## Delivery times` |
| List items on their own line | `- Free returns within 30 days` |
| Table rows: one line per row, with column labels | `Part: NMEF 4/22; AWG: 22; OD: 0.162` |
| PDF pages: an invisible page marker before each page's text | (character U+0001 + page number) |

- **Posts and pages** (PHP, `ContentExtractor::extract()`): runs `the_content` filters (blocks, shortcodes), drops
  scripts/styles/forms, turns tables into labelled rows, headings into `## …`, block tags into blank lines.
- **Files** (browser, `extractors/`): PDF text page by page (`pdf.ts`, keeps page numbers for `file.pdf#page=N`
  links), DOCX through mammoth → HTML → same cleanup (`docx.ts`), TXT/MD as they are (`text.ts`).
- **Normalization** (`normalizeText()`, mirrors PHP): same line breaks, spaces and Unicode form every time, so the
  text only changes when the words change (important for the hashes in step 4).

Why table rows carry their labels: a row like `22 | 0.162` means nothing once the chunker separates it from the
header row. `AWG: 22; OD: 0.162` still makes sense alone.

The format has a version: `ContentExtractor::TEXT_FORMAT = 3`. Changing the rules bumps it, which marks every indexed
item outdated so it is re-chunked.

---

## 3. Step 2 — The chunker (`chunkText()`)

```ts
export function chunkText(text: string, options: ChunkerOptions): TextChunk[] {
  const sections = parseSections(text);                                      // 2a
  const chunks = [];
  for (const section of sections) {
    for (const { content, page } of packSection(section.paragraphs, options.targetWords, options.overlapWords)) { // 2b–2d
      for (const piece of fitTokens(content, section.heading, page, options)) chunks.push(piece);                 // 2e
    }
  }
  return [...outlineChunks(sections, options), ...mergeTiny(chunks, options)];   // 2f, 2g
}
```

Settings (Settings → Advanced): **chunk size** `chunk_words` = 250 words (allowed 100–350), **overlap**
`chunk_overlap` = 40 words. Model limit: **maxTokens** = 512 for bge-small.

### 2a. Split at headings (`parseSections`)

Every `## Heading` paragraph starts a new **section**. Chunks never cross a heading.

Why: an edit in one section changes only that section's chunks. Other sections keep the same text, the same hash,
and their old vectors (no re-embedding).

The PDF page marker is read here too: every paragraph remembers the page it is on.

### 2b. Break up long paragraphs (`splitLong`)

A paragraph longer than the chunk size is cut, in this order of preference:

1. **By line**, when it has several lines (a table or list): whole rows / items are packed, a line is never cut
   while it fits (`splitLines`).
2. **By sentence** (after `.` `!` `?`).
3. **By word**, only for a single "sentence" longer than the chunk size (rare: long lists without punctuation).

### 2c. Pack paragraphs into chunks (`packSection`)

Whole paragraphs are added to the current chunk until the next one would pass ~250 words; then the chunk is closed
and a new one starts. Paragraphs inside a chunk stay separated by a blank line.

A chunk's **page number** is the page where its own new text starts (the overlap from the previous chunk doesn't
count), so the citation link opens the right PDF page.

### 2d. Overlap (`overlapOf`)

Each new chunk starts with the end of the previous chunk of the same section:

- normal text: the **last 40 words**;
- after a table or list: **whole trailing lines** up to 40 words (a row is never cut in half).

Why: a sentence that answers a question may sit right at a chunk boundary. With overlap, it is complete in at least
one chunk.

```
section text:  ……… A ……… B ……… C ………
chunk 1:       [ A ……… B ]
chunk 2:            [ last 40 words of chunk 1 | C ……… ]
```

### 2e. Fit the model's token limit (`fitTokens`)

The model reads at most **512 tokens**; anything longer would be silently cut off and its end lost. The text sent to
the model is not the chunk alone but **"Title › Heading › chunk text"** (`modelInputFor()`), and it is counted with
the model's own tokenizer.

If it doesn't fit, the chunk is **halved**, between lines when it has several (rows, items, paragraphs), else between
words, and each half is checked again (recursively) until every piece fits.

250 words are usually ~330–400 tokens, so this mostly triggers on dense tables, codes and numbers (more tokens per
word).

### 2f. Merge tiny chunks (`mergeTiny`)

A chunk under **5 words** (e.g. a stray "Dev" label from a page builder) joins the previous chunk (or the next one,
when it is first), with its heading inline: `Price: $5`. It stays alone only if merging would break the token limit.

Why: a tiny chunk's vector is almost only the page title. It scored 0.676 for "What industry do you serve?" without
holding any answer and pushed real sections out of the top results.

### 2g. Outline chunk (`outlineChunks`)

A page with **3 or more headings** gets one extra chunk first: `Page title: Heading A, Heading B, Heading C`.

Why: questions like "Which industries do you serve?" are answered by the list of sections, which no single section
contains. Repeating the page title in it scored best in tests (0.79 vs 0.71).

---

## 4. Step 3 — Hash, reuse, embed (`pipeline.ts`)

For each chunk:

1. **chunk_hash** = `sha256( model key + "|" + "Title › Heading › text" )`. The model key contains model id, revision,
   dtype, pooling and normalize (`EmbeddingModel::hash_key()`).
2. **Reuse:** if the resource already has a chunk with the same hash (`knownHashes` from the server), its stored
   vector is kept and nothing is embedded. Editing one paragraph re-embeds only the chunks around it.
3. **Embed** the rest in batches:

```ts
const result = await embedder.embed(batch.map((i) => chunks[i].modelInput));   // Float32Array(384) each
if (result[j]?.length !== config.model.dimensions) throw new Error(`… expected ${config.model.dimensions}.`);
```

| Provider | Who makes the vector |
|---|---|
| Local (bge-small-en-v1.5, q8) | Transformers.js in the admin's browser (Web Worker) |
| API (OpenRouter) | The browser sends chunk texts to `POST /jobs/embed`; PHP calls OpenRouter, packs the numbers with `VectorCodec::encode()` (`pack('g*')`) and returns them |

Either way the chunks are made in the browser, and the browser sends the finished job back.

---

## 5. Step 4 — Store

The browser sends every chunk to `POST /jobs/{id}/complete`: text, heading, page, token count, hash, and the vector
as **base64** of its raw bytes (`float32ToBase64()`), or `null` = "reuse the stored vector with this hash".

PHP (`IndexingService`) decodes it (`VectorCodec::from_base64()`), checks the size (bytes ÷ 4 must equal the model's
dimensions), and stores the bytes unchanged in `wp_mnaisp_chunks.embedding`. PHP never changes a local vector.

```
Float32Array(384)  →  raw bytes (1,536, little-endian)  →  base64 (2,048 chars)  →  PHP  →  BLOB (1,536 bytes)
```

The stored format (float32 little-endian) is the same as MariaDB's `VECTOR(384)`: the BLOBs can be copied into a
vector column as they are (tested 2026-11-04).

---

## 6. Worked example

Page "Shipping Policy" (title), 3 sections:

```
Our shipping rules.                                     ← no heading, 3 words
## Delivery times                                      ← heading
Delivery takes 3–5 days… (320 words, 4 paragraphs)
## Costs
| Zone | Price |  → "Zone: EU; Price: $5" / "Zone: US; Price: $9"   ← table rows
## Returns
Free returns within 30 days… (90 words)
```

| # | Chunk | Why |
|---|---|---|
| 0 | `Shipping Policy: Delivery times, Costs, Returns` | Outline: 3 headings |
| 1 | "Our shipping rules." + Delivery times, paragraphs 1–3 (~240 words) | The 3-word intro is under 5 words: it has no chunk before it, so it joins the next one |
| 2 | Delivery times, last 40 words of #1 + paragraph 4 | Packed to ~250, then overlap |
| 3 | Costs: `Zone: EU; Price: $5` / `Zone: US; Price: $9` | Own section, rows intact |
| 4 | Returns (90 words) | Own section |

Each chunk gets a 1,536-byte vector, short or long. Editing only the Returns text later re-embeds only chunk 4;
chunks 0–3 keep their hashes and vectors. Renaming a heading also changes the outline chunk 0. Changing the page
title changes every chunk (the title is in every model input).

---

## 7. Glossary

- **Chunk:** a small piece of a page's text, searched and cited on its own.
- **Section:** text between two headings. Chunks never cross one.
- **Chunk size / overlap:** target words per chunk (250) and words repeated from the previous chunk (40).
- **Token:** the model's unit of text, a word or part of a word. Numbers and codes use more tokens per word.
- **maxTokens:** the longest input the embedding model reads (512 for bge-small). Longer input would be cut off.
- **Model input:** the exact text embedded, "Title › Heading › chunk text". Never stored; rebuilt any time.
- **Outline chunk:** extra first chunk listing a page's headings, for "which … do you offer" questions.
- **chunk_hash:** fingerprint of model key + model input. Same hash = same vector, so it is reused.
- **Vector / embedding:** the model's output, 384 numbers that describe the meaning. Fixed size per model.
- **float32 / little-endian:** each number takes 4 bytes, stored lowest byte first. 384 × 4 = 1,536 bytes.
- **TEXT_FORMAT:** version of the text-cleanup rules; bumping it marks the index outdated.
