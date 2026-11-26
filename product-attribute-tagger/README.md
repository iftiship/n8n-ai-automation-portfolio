# Product Attribute Tagger — Vision Merchandising Engine

## Workflow Preview
<img width="1875" height="862" alt="Screenshot 2026-11-26 104308" src="https://github.com/user-attachments/assets/f8bc6e24-9d9e-4ac8-ae59-a269d856e9da" />

> **Design Principle:** Validate first, extract with Vision AI + automatic fallback, and keep a human in the loop whenever confidence drops below 80%.

## The Problem It Solves
Every season, merchandising teams have to turn hundreds of garment studio photos into structured catalog rows — colour, hex code, fabric, weight, weave/texture, neckline, sleeve style, occasion, care instructions, and SEO tags.

Doing this by hand is slow and inconsistent (`"Deep Red"` in one row, `"Crimson"` in the next). On the other hand, naive AI scripts waste money re-tagging duplicate SKUs, hallucinate tags for accidental screenshots, and push low-confidence guesses straight into the live catalog.

I built this **n8n Vision AI pipeline** to solve both: it catalogs garment photos automatically into **Google Sheets**, protects against bad files and duplicate SKUs, and pauses for **1-tap Telegram approval** whenever the AI isn't 80%+ sure.

---

## How It Works (In Plain English)

1. **Dual-Channel Intake (`Drive Garment Intake` / `Webhook Garment Intake`)**
   - Drop a garment photo into the Google Drive intake folder (`01_Intake_Garments`) **or** send a request to `POST /webhook/product-attribute-tagger` (JSON with `image_url`, or binary upload).
2. **Pre-Flight Check & Duplicate Guard (`Pre-Flight & Normalize` → `Check Existing SKU`)**
   - Verifies file type (`JPEG`, `PNG`, `WEBP`, `GIF`) and size (`1 KB – 15 MB`). Bad payloads are logged to `Error_Audit_Log` and return `400`.
   - Checks `Catalog_Attributes` by `sku_id` first (while keeping the image binary intact) so duplicate SKUs return `200 duplicate_skipped` without burning Vision AI credits.
3. **Dual Vision AI & Normalization (`Gemini Vision` + `OpenAI Vision` Fallback)**
   - Runs **Gemini Vision** (`gemini-3-flash-preview`) first; if it fails, automatically routes the image to **OpenAI `gpt-4o-mini` Vision**.
   - `Parse & Normalize Attributes` cleans up the JSON, validates hex color codes, and calculates a `confidence_score` (0–100).
4. **Quality Gate, Sheets & Human Approval (`Valid Garment?` → `High Confidence (>=80%)?`)**
   - **Not a garment?** Screenshots or documents are rejected (`422`) and logged to `Error_Audit_Log`.
   - **Confidence ≥ 80%:** Upserts into `Catalog_Attributes`, sends a formatted Telegram summary card, returns `200`, and moves Drive files to `02_Processed_Cataloged`.
   - **Confidence < 80% (or `require_hitl: true`):** Logs to `Needs_Review`, returns `202 pending_hitl_approval`, and waits for a merchandiser's **1-tap `Approve / Reject`** on Telegram before promoting or rejecting the SKU.
5. **Error Safety Net (`On Workflow Error` → `Alert Owner on Telegram`)**
   - Captures any unexpected failure and sends a Telegram alert with a direct link to the failed run.

---

## Google Sheets Tabs (`Master Catalog`)

| Sheet Tab | Purpose |
|---|---|
| `Catalog_Attributes` | Live approved SKUs with 24 structured attributes (colour, hex, fabric, texture, design, silhouette, occasion, care, SEO tags, confidence, provider, timestamp) |
| `Needs_Review` | Low-confidence or HITL-flagged SKUs awaiting merchandiser sign-off + `review_reason` |
| `Error_Audit_Log` | Rejected files, non-garment uploads, and failure diagnostics with `execution_id` |

---

## Tech Stack
- **Orchestration:** n8n (31 functional nodes, 5 color-coded stages)
- **Vision AI:** Google Gemini Vision (`gemini-3-flash-preview`) + OpenAI `gpt-4o-mini` Vision fallback
- **Storage & Intake:** Google Drive, Google Sheets
- **Human-in-the-Loop & Alerts:** Telegram Bot API

## Quick Setup
1. Import [`workflow.json`](./workflow.json) into n8n.
2. Connect your **Google Drive**, **Google Sheets**, **Google Gemini**, **OpenAI** (or n8n Gateway), and **Telegram** credentials.
3. Update the Drive folder IDs, Google Sheet ID, and Telegram Chat ID to match your workspace.
