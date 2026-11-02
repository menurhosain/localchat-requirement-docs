# AI Engine (Jordy Meow / Meow Apps)

- URL: https://wordpress.org/plugins/ai-engine/
- Fetched: 2026-10-27
- Version 3.8.8 · since 2022-12 · 90,000+ installs · 4.9★ (872 reviews)
- WP 6.0+, PHP 8.1+ · tested WP 7.1.3

## What it does
Market leader. General AI toolkit for WP: chatbots, admin Workspace chat (+ mobile apps),
Content/Image Studio, AI Forms, embeddings/KB, function calling, moderation,
MCP server (130+ tools free) letting Claude/ChatGPT operate the site.

## Knowledge base (PRO only)
- Posts synced to KB, PDF import with auto-chunking
- Vector DB: Pinecone, Qdrant, Chroma, OpenAI Vector Store. NO built-in local store mentioned.
- Search modes: Simple, Context-Aware, Smart. Auto-sync, page-by-page sync for big stores.
- Citations: not mentioned. FAQ admits can't fully block model's general knowledge.

## Free vs Pro
Free: chatbots, Workspace, studios, MCP server. Pro: embeddings + vector DBs, content-aware,
function calling, cross-site bots, realtime audio, stats/limits.

## AI providers
OpenAI, Anthropic, Google, xAI, Mistral, Perplexity, OpenRouter, Replicate, Azure, OpenAI-compatible (Ollama, LM Studio, vLLM).

## Data
Self-hosted, BYO key, no markup. Free version sends nothing to Meow Apps.

## Strengths
- Huge trust: 90k installs, 872 reviews, very active
- Every provider incl. local models
## Weaknesses (for our goal)
- KB/RAG is Pro AND needs external vector DB service (Pinecone etc.) — setup heavy for non-tech users
- No citations, no strict "answer only from content"
- PHP 8.1+, caching/firewall conflicts, recent security fixes
- General toolkit, KB is one feature among many
