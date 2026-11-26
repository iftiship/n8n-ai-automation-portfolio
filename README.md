# n8n AI & Retail Automation Portfolio

Production-grade **n8n** workflows engineered with structured JSON outputs, pre-flight validation, automatic LLM fallback routing, human-in-the-loop (HITL) approvals, and dead-letter error monitoring.

## Flagship Automations

| # | Project | Stack | Status |
|---|---|---|---|
| 1 | **[Product Attribute Tagger](./product-attribute-tagger/)** | n8n, Gemini 3 Flash Vision, OpenAI `gpt-4o-mini` Fallback, Google Drive, Google Sheets, Telegram HITL | **Live & Production-Ready** |
| 2 | **[Retail Support RAG Agent](./retail-support-rag-agent/)** | n8n, Supabase `pgvector`, OpenAI `text-embedding-3-small`, Gemini 2.5 Flash + `gpt-4o-mini` Fallback, Telegram Escalation | **Live & Production-Ready** |
| 3 | **[Store Inventory & Coaching Agent](./store-inventory-coaching-agent/)** | n8n, Supabase, Gemini 2.5 Flash + `gpt-4o-mini` Fallback, Telegram HITL Transfer Approvals | **Live & Production-Ready** |
| 4 | **Ask-Your-Sales-Data Agent** | n8n, Natural-Language Querying, Supabase, LLM | *Coming Next* |

---

## Reliability & Engineering Standards
- **Structured JSON Enforcement:** Strict schema parsing and taxonomy normalization.
- **Pre-Flight & Idempotency Gates:** MIME/size validation and SKU deduplication before incurring LLM costs.
- **Automatic Fallback Routing:** Primary model failures route seamlessly to secondary vision/LLM providers.
- **Human-in-the-Loop (HITL):** Low-confidence or flagged items pause for 1-tap Telegram approval before catalog promotion.
- **Error Monitoring:** Dedicated `errorTrigger` safety net with audit logging and instant Telegram alerts.
