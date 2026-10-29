# Webhook Ingestion & Payload Mapping Engine — Review Artifact

This directory contains the interactive prototype and documentation designed for team review based on our audio requirements discussion.

## Files in this Directory

* **[`index.html`](file:///D:/Nature/Artifact/index.html)**: Standalone, zero-install interactive UI prototype. Double-click to open in any web browser (Chrome, Edge, Firefox, Safari).
* **[`README.md`](file:///D:/Nature/Artifact/README.md)**: Overview and review guide for the team.

---

## What the Interactive UI Demonstrates

1. **Visual Mapper Studio**:
   * **Source Payload Panel**: Load pre-configured sample payloads (Shopify, Facebook Lead Ads, Custom Partner CRM) or paste custom JSON.
   * **Dynamic Mapping Rules**: Map nested fields (e.g. `customer.first_name` + `customer.last_name` $\rightarrow$ `name`, `customer.email` $\rightarrow$ `email`). Supports transforms (concatenation, uppercase, lowercase, numeric casting) and fallback defaults.
   * **Live Canonical Output**: Real-time transformation preview rendering the standardized entity (e.g. `Lead`, `Customer`) ready for database persistence.

2. **Dynamic Endpoint Management (Pathway 3)**:
   * Create dedicated webhook URLs for external platforms (e.g. `/api/v1/webhooks/wh_shopify_leads_9f2a`).
   * Binds each third-party provider to its own saved mapping configuration and HMAC security validation.

3. **3 Ingestion Pathways Comparison**:
   * **Pathway 1 (Canonical)**: Standard `/api/v1/ingest/canonical` for pre-formatted internal payloads.
   * **Pathway 2 (In-Payload)**: `/api/v1/ingest/dynamic` where callers submit data alongside an inline `mapping` schema block.
   * **Pathway 3 (Dynamic Endpoints)**: Dedicated URLs per provider with stored database mapping rules.
