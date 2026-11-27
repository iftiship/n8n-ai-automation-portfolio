# Retail Support RAG Agent — Policy, Order Lookup & Escalation Engine

**Design Principle:** Ground every answer in the store's own policy documents, switch to a backup AI model if the first one fails, and hand off to a human agent whenever confidence drops below 70%.

## Workflow Preview
<img width="1880" height="830" alt="IMG_20261126_193915" src="https://github.com/user-attachments/assets/3505ec65-a216-47f8-9594-5f10082559f9" />

## The Problem It Solves
During wedding and festive seasons, ethnic-wear support teams get flooded with the same repetitive questions: *"Can I exchange my sherwani?", "How long do alterations take?", "Where is my order?"*

Generic chatbots fail here in two ways:
1. **They hallucinate** — making up return windows or refund promises that violate store policy.
2. **They trap angry customers in loops** — failing to hand off sensitive cases (like a damaged custom bridal lehenga) to a real person.

I built this **n8n + Supabase** agent to solve both: it answers policy and live order questions **only** from verified store data, and immediately escalates tricky or low-confidence cases to a human specialist on Telegram.

---

## How It Works (In Plain English)

1. **Knowledge Base Ingestion (`KB Webhook`)**
   - Store policies (returns, sizing, alterations, fabric care, VIP appointments) are chunked, embedded with OpenAI `text-embedding-3-small`, and stored in **Supabase `pgvector`** (`customer_kb`).
2. **Multi-Channel Intake (`Support Chat` & `Support Webhook`)**
   - Customers can ask questions via live web chat or a REST API. `Normalize Input` standardizes the session ID, contact info, and question.
3. **Grounded AI Agent (`RAG Agent`)**
   - **Policy Lookup (`KB Retriever`):** Searches `customer_kb` before answering any policy question.
   - **Live Order Lookup (`Order Lookup`):** Queries the Supabase `orders` table by `order_id` (e.g., `MNY-10421`) for real-time shipping status and tracking numbers.
   - **Reliability:** Runs on **Gemini 2.5 Flash** with automatic fallback to **OpenAI `gpt-4o-mini`**, `Chat Memory` for multi-turn context, and a `JSON Parser` that outputs a `confidence_score` (0–100).
4. **Confidence Gate & Human Escalation (`Confidence Gate` → `Escalate?`)**
   - Every conversation is logged to Supabase (`support_tickets`) for auditability.
   - **Confidence ≥ 70%:** Replies directly to the customer (`status: "resolved"`).
   - **Confidence < 70% or Policy Exception:** Sends an instant alert card to a human specialist on **Telegram** (`Telegram Escalate`) with the customer's details, question, and AI draft, then returns `status: "escalated_to_human"`.
5. **Error Safety Net (`Error Trigger` → `Error Alert`)**
   - Any unexpected runtime failure immediately alerts Telegram with a direct link to the failed execution.

---

## Tech Stack
- **Orchestration:** n8n (29 nodes, 5 modular stages)
- **LLMs & Embeddings:** Google Gemini 2.5 Flash (Primary), OpenAI `gpt-4o-mini` (Fallback), OpenAI `text-embedding-3-small`
- **Database & Vector Store:** Supabase (`pgvector` `customer_kb`, `orders`, `support_tickets`)
- **Human Escalation & Alerts:** Telegram Bot API

## Quick Setup
1. Import [`workflow.json`](./workflow.json) into n8n.
2. Create `customer_kb` (`match_customer_kb` RPC), `orders`, and `support_tickets` in Supabase.
3. Connect your **Supabase**, **Google Gemini**, **OpenAI** (or n8n Gateway), and **Telegram** credentials.
4. Seed policies via `POST /webhook/retail-kb-ingest` and test queries via `POST /webhook/retail-support-agent`.
