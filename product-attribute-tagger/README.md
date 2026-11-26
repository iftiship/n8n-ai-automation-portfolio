# Product Attribute Tagger — Vision Merchandising Engine

A production-ready, vision-based garment cataloging pipeline built in **n8n** that reads garment product images from **Google Drive** or a **REST Webhook**, extracts structured merchandising attributes (**colour, hex, fabric, weight, texture, design type, silhouette, and SEO tags**), and upserts them directly into **Google Sheets** with automatic fallback routing and **Telegram HITL** quality assurance.

## Architecture Overview

1. **Dual-Channel Intake**
   - **Google Drive Watcher:** Polls `01_Intake_Garments` every minute and downloads newly uploaded garment images.
   - **REST Webhook (`POST /webhook/product-attribute-tagger`):** Accepts JSON (`{ sku_id, product_title, image_url, require_hitl }`) or `multipart/form-data` binary uploads.
2. **Pre-Flight Validation & Deduplication**
   - Validates image MIME types (`JPEG`, `PNG`, `WEBP`, `GIF`) and file size (`1 KB – 15 MB`). Invalid payloads are logged to `Error_Audit_Log` and return `400`.
   - Checks `Catalog_Attributes` by `sku_id` while preserving the binary image so duplicate SKUs return `200 duplicate_skipped` without re-running Vision LLMs.
3. **Dual Vision AI & Fallback**
   - **Primary:** `Gemini 3 Flash Vision` (`models/gemini-3-flash-preview`, 2x retry).
   - **Fallback:** On API error, automatically routes the image binary to `OpenAI GPT-4o-mini Vision`.
   - `Parse & Normalize Attributes` enforces structured JSON, validates hex codes, and computes confidence routing.
4. **Semantic Gate, Sheets & HITL**
   - **Non-Garment Rejection (`422`):** Rejects non-apparel images (e.g., documents, screenshots) and logs to `Error_Audit_Log`.
   - **High Confidence (`>= 80%`):** Upserts into `Catalog_Attributes`, sends a formatted Telegram catalog card, responds `200`, and moves Drive files to `02_Processed_Cataloged`.
   - **Low Confidence (`< 80%` or `require_hitl: true`):** Logs to `Needs_Review`, responds `202 pending_hitl_approval`, and pauses for 1-tap Telegram Merchandiser Approval (`✅ Approve` / `❌ Reject`).
5. **Production Error Monitor**
   - `On Workflow Error` (`errorTrigger`) captures any uncaught failure and alerts Telegram with a direct execution link.

## Google Sheets Schema (`Master Catalog`)
- **`Catalog_Attributes`:** `sku_id`, `product_name`, `garment_category`, `primary_colour`, `secondary_colours`, `colour_family`, `hex_estimate`, `fabric_material`, `fabric_weight`, `texture_finish`, `design_type`, `pattern_motif`, `neckline_collar`, `sleeve_style`, `fit_silhouette`, `occasion_season`, `care_instructions`, `merchandising_tags`, `confidence_score`, `vision_provider`, `source_type`, `source_ref`, `status`, `processed_at`
- **`Needs_Review`:** Same schema + `review_reason`
- **`Error_Audit_Log`:** `logged_at`, `sku_id`, `source_type`, `source_ref`, `error_stage`, `error_reason`, `fallback_attempted`, `execution_id`

## Setup
1. Import [`workflow.json`](./workflow.json) into your n8n instance.
2. Connect your **Google Drive**, **Google Sheets**, **Google Gemini**, **OpenAI** (or n8n Gateway), and **Telegram** credentials.
3. Update the Drive folder IDs, Google Sheet ID, and Telegram Chat ID to match your workspace.
