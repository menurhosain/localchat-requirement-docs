# AI Chatbot Free Models (NewCodeByte)

- URL: https://wordpress.org/plugins/chatbot-ai-free-models/
- Fetched: 2026-10-27
- Version 2.0.7 · since 2025-03 · 100+ installs · 5★ (10 reviews)
- WP 5.0+, PHP 7.4+ · tested WP 7.1.3 · GPLv2 · solo developer

## What it does
Self-hosted lightweight chatbot widget (floating or `[newcodebyte_chatbot]` shortcode).
Bring your own API key; use free models (OpenRouter `:free`) for zero cost.
All config in WP admin. Standalone non-WP version on GitHub.

## Features (all free)
Multi-provider; appearance (colors, position, height, images, sounds); streaming w/ Stop + Retry;
pre-chat form (name, email); proactive welcome bubble; typing indicator; live KB search of posts/pages
with cited source URLs; admin Messages tab (search, read, delete, export TXT/CSV/MD/HTML);
unread badge; per-IP rate limit; optional "Powered by" link.

## AI / LLM
OpenAI, Anthropic, Google, DeepSeek, Groq, xAI, Together, Mistral, Perplexity, OpenRouter (500+),
any OpenAI-compatible endpoint. Reviewer: GPT OSS 120B free follows prompts well; small models less.

## Knowledge sources
Posts + pages only, via one SQL keyword query per question (title + body). No copy/storage.
Plus free-text "Information for responses" field. NO PDFs, URLs, Q&A, embeddings, vector DB.
Context budget default 3,500 tokens (filter).

## Pricing
Plugin 100% free, donations. Pay only AI provider.

## Data
Self-hosted. Chat saving off by default (separate DB table). Messages go to chosen provider.
Privacy note says OpenRouter only — conflicts w/ multi-provider.

## Strengths
- Zero cost possible, no SaaS, no lock-in
- Simple, transparent, cites sources
- Huge provider choice incl. custom endpoints

## Weaknesses
- Naive retrieval (SQL keyword match, not semantic)
- No docs/PDF, no handoff, no Woo, no channels
- Free models slow/unreliable; proxy config needed for rate limit
