# Store Inventory & Coaching Agent

> **Design Principle:** Code computes the signals. AI explains them. A human approves any stock movement.

## The Problem It Solves
In multi-store ethnic-wear retail, two things quietly hurt revenue every week:
1. **Imbalanced Store Stock:** One flagship store sells out of a bestselling Sherwani right before wedding weekend, while another city has 11 units sitting unsold.
2. **Spreadsheet Fatigue:** Store managers get raw inventory dumps instead of clear daily priorities (*"Move 4 units from Bengaluru to Delhi today; push these slow-moving jackets"*).

I built this **16-node n8n + Supabase** agent to spot stockout risks and slow movers automatically, propose inter-store transfers with **1-tap Telegram approval**, and send daily coaching briefs to store managers.

---

## How It Works (In Plain English)

1. **Trigger & Snapshot (`Daily Schedule` / `Ops Webhook` → `Fetch Store Stock & Sales`)**
   - Runs every morning at 8:30 AM IST (or on demand via `POST /webhook/store-ops-run`) and pulls live stock and 7-day sales velocity from Supabase (`store_inventory`).
2. **Signals & Decision (`Compute Signals` → `Needs Action?`)**
   - Flags **Stockout Risks** (`days_of_cover < 5` or below reorder point) and **Slow Movers** (`days_of_cover > 30` with ≤ 1 sale in 7 days).
   - Automatically pairs overstocked donor stores with needy stores for the same SKU and size — and **skips the AI completely** if all stores are healthy.
3. **AI Coaching (`Ops Coach Agent`)**
   - Runs on **Gemini 2.5 Flash** with automatic fallback to **OpenAI `gpt-4o-mini`** and a `JSON Parser`.
   - Outputs the top inter-store transfer recommendation (with urgency & business rationale) plus a sharp daily coaching brief.
4. **Human Approval (`Log Transfer (Pending)` → `Transfer HITL` → `Update Transfer`)**
   - Logs the transfer request in Supabase (`transfer_requests` as `pending_approval`).
   - Sends the coaching brief to Telegram (`Send Coaching to Manager`), replies `200` to any webhook caller, and pauses for **1-tap `Approve / Reject`** on Telegram before updating Supabase.
5. **Error Monitor (`Error Trigger` → `Error Alert`)**
   - Alerts Telegram immediately with a direct execution link if anything fails.

---

## Tech Stack
- **Orchestration:** n8n (16 functional nodes, 5 color-coded stages)
- **LLMs:** Google Gemini 2.5 Flash (Primary), OpenAI `gpt-4o-mini` (Fallback)
- **Database:** Supabase (`store_inventory`, `transfer_requests`)
- **Coaching & HITL:** Telegram Bot API (`sendMessage` + `sendAndWait` approval)

## Quick Setup
1. Import [`workflow.json`](./workflow.json) into n8n.
2. Create `store_inventory` and `transfer_requests` tables in Supabase.
3. Connect your **Supabase**, **Google Gemini**, **OpenAI** (or n8n Gateway), and **Telegram** credentials.
4. Trigger via `POST /webhook/store-ops-run` or let the 8:30 AM IST schedule run daily.
