# Ask-Your-Sales-Data Agent — Guardrailed Retail BI Engine

## Workflow Preview
<img width="1870" height="867" alt="IMG_20261126_224550" src="https://github.com/user-attachments/assets/a41e2fc4-2ea9-4c86-bdcc-78463dc3da58" />

> **Design Principle:** The LLM plans the question and narrates the insight. Deterministic code enforces security, redacts PII, and does all the math.

## The Problem It Solves
Retail founders and store heads constantly ask ad-hoc business questions: *"Which store had the highest Sherwani velocity this week?"* or *"How many orders are still pending dispatch?"*

Usually, they either wait days for an analyst to pull a CSV, or they plug an AI chatbot straight into their database — which creates three dangerous risks:
1. **Prompt & SQL Injection:** A bad prompt could try to drop tables, mutate records, or leak customer PII.
2. **Hallucinated Math:** LLMs are notoriously unreliable at summing revenue, computing days-of-cover, or ranking SKUs.
3. **Channel Friction:** Leaders want answers inside **Slack, WhatsApp, Web Chat, or an API** — not another BI dashboard login.

I built this **36-node omnichannel n8n + Supabase BI agent** to solve all three: it accepts natural-language questions from any channel, enforces strict read-only guardrails, computes exact KPIs in deterministic code, and narrates only verified numbers back to the channel you asked from.

---

## How It Works (In Plain English)

1. **Omnichannel Intake & Guard (`Sales BI Chat` / `Sales BI Webhook` / `Slack BI Trigger` / `WhatsApp BI Trigger` → `Normalize & Input Guard` → `Input Safe?`)**
   - Listens across **Web Chat, REST Webhook, Slack, and WhatsApp**, filters out bot loops, and blocks prompt/SQL injection before any LLM is called.
2. **Query Planner (`Query Planner Agent` — Dual-LLM)**
   - Uses **Gemini 3 Flash** (with automatic fallback to **OpenAI `gpt-4o-mini`**), `Session Memory`, and `Plan JSON Parser` to translate plain English into a structured, read-only query plan.
3. **Security Gate & Deterministic BI Engine (`Query Security Gate` → `Deterministic Analytics Engine`)**
   - Enforces **SELECT-only** rules, routes to `orders` or `store_inventory` in Supabase, **redacts PII**, and computes all totals, averages, and rankings deterministically in JavaScript.
4. **Executive Narrator & Channel-Native Replies (`Executive Narrator Agent` → `Route Reply Channel`)**
   - Runs **DeepSeek Chat** (with fallback to **OpenAI `gpt-4o-mini`**) to turn verified KPIs into a crisp executive summary, then replies natively to **Webhook (`200`), Slack, WhatsApp, or Web Chat**.
5. **Production Error Monitor (`Error Trigger` → `Error Alert`)**
   - Captures any uncaught failure and alerts **Telegram** immediately.
6. **Guardrail Refusals & Alerts (`Build Guardrail Refusal` → `Security Threat?` → `Route Refusal Channel`)**
   - Sends an instant **Telegram** security alert on blocked threats (`403`) and returns a safe refusal on the origin channel.

---

## Tech Stack
- **Orchestration:** n8n (36 functional nodes, 6 color-coded stages)
- **LLMs:** Google Gemini 3 Flash (Planner), DeepSeek Chat (Narrator), OpenAI `gpt-4o-mini` (Fallback for both)
- **Database:** Supabase (`orders`, `store_inventory`)
- **Channels & Alerts:** Web Chat, REST Webhook, Slack, WhatsApp, Telegram (Security & Error Alerts)

## Quick Setup
1. Import [`workflow.json`](./workflow.json) into n8n.
2. Connect your **Supabase**, **Google Gemini**, **DeepSeek**, **OpenAI** (or n8n Gateway), **Slack**, **WhatsApp**, and **Telegram** credentials.
3. Publish the workflow and ask a question from Web Chat, Slack, WhatsApp, or the REST Webhook.
