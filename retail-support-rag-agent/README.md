# Retail Support RAG Agent — Policy & Escalation Engine

A production-ready, grounded **Retrieval-Augmented Generation (RAG)** customer support agent built in **n8n** and **Supabase `pgvector`**. It ingests retail policy and ethnic-wear catalog documents, answers customer inquiries over **Web Chat** or **REST Webhook** with strict knowledge-base grounding, logs every ticket to **Supabase**, and automatically escalates low-confidence (`< 70%`) or policy-exception cases to a human specialist on **Telegram**.

## Architecture Overview

1. **Vector Knowledge Base Ingestion (`POST /webhook/retail-kb-ingest`)**
   - `Split KB Documents` normalizes incoming policy/catalog JSON arrays (`{ documents: [{ topic, category, text }] }`).
   - `Embeddings OpenAI (Ingest)` (`text-embedding-3-small`, 1536 dimensions) + `KB JSON Data Loader` embed and upsert chunks into Supabase `public.customer_kb` (`match_customer_kb` RPC).
   - Returns `200` with `{ ok: true, status: "ingested", chunks_added }`.
2. **Multi-Channel Support Intake**
   - **Web Chat (`Customer Support Chat`):** Native n8n AI chat interface.
   - **REST Webhook (`POST /webhook/retail-support-agent`):** Accepts `{ question, session_id, customer_name, customer_contact }`.
   - `Normalize Customer Inquiry` standardizes channel, session ID, customer metadata, and query text.
3. **Grounded RAG Agent + Fallback LLM**
   - **Primary LLM:** `Gemini 2.5 Flash` (`models/gemini-2.5-flash`, temperature `0.2`).
   - **Fallback LLM:** `OpenAI GPT-4o-mini` (`needsFallback: true`) on primary provider failure.
   - **Vector Retrieval Tool:** `Retail Knowledge Base` queries Supabase `customer_kb` (`topK: 5`, `match_customer_kb`) using `text-embedding-3-small`.
   - **Session Memory:** `Session Memory` (`memoryBufferWindow`, window `6`, keyed by `session_id`).
   - **Structured Output:** `Structured Support Parser` enforces JSON (`answer`, `confidence_score`, `intent_category`, `needs_human_escalation`, `escalation_reason`, `cited_policy_topic`).
4. **Confidence Gate, Ticket Audit & Human Escalation**
   - `Evaluate Confidence & Escalation` flags any response with `confidence_score < 70`, `needs_human_escalation === true`, or missing answer.
   - `Log Ticket to Supabase` records every interaction in `public.support_tickets`.
   - **Resolved (`confidence >= 70%`):** Returns `200` with `status: "resolved"` and `ticket_id`.
   - **Escalated (`confidence < 70%` or exception):** Sends an instant HTML alert card to **Telegram** (`Escalate to Human (Telegram)`) and returns `200` with `status: "escalated_to_human"`.
5. **Production Error Monitor**
   - `On Workflow Error` (`errorTrigger`) captures uncaught runtime failures and alerts Telegram with a direct execution link.

## Supabase Schema
- **`public.customer_kb`:** `id`, `content`, `metadata` (`topic`, `category`), `embedding` (`vector(1536)`) + `public.match_customer_kb` RPC.
- **`public.support_tickets`:** `id`, `created_at`, `session_id`, `channel`, `customer_name`, `customer_contact`, `question`, `answer`, `intent_category`, `confidence_score`, `escalated`, `escalation_reason`, `status`.

## Setup
1. Import [`workflow.json`](./workflow.json) into your n8n instance.
2. Run the Supabase migration for `customer_kb`, `match_customer_kb`, and `support_tickets`.
3. Connect your **Supabase**, **Google Gemini**, **OpenAI** (or n8n Gateway), and **Telegram** credentials.
4. Seed policies via `POST /webhook/retail-kb-ingest` and query via `POST /webhook/retail-support-agent`.
