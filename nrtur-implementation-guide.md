# Nrtur Lead Ingestion & Dynamic Webhook Engine
## Full Codebase Analysis (Backend & Frontend) & Technical Specification

---

## 1. Executive Summary & Core Architectural Rule

After a complete inspection of both **client-side** (`d:\Nature\nrtur-frontend`) and **server-side** (`d:\Nature\nrtur-backend`) codebases:

> [!IMPORTANT]
> **Core Architectural Finding: Single `name` Field for Leads**
> In Nrtur:
> * **Contacts** (`/contacts`) accept `first_name` and `last_name` separately.
> * **Leads** (`/leads` and `/api/public/v1/leads`) accept **only a single `name` field** (`dto.PublicCreateLeadRequest.Name` / `dto.CreateLeadRequest.Name`).
>
> External webhook senders (such as Shopify's `customer.first_name` + `customer.last_name`, or Meta Lead Ads' form fields) send first and last names separately. Because Nrtur's Lead backend rejects separate first and last name fields, **the mapping engine must combine/map them into Nrtur's single `name` string** before executing `CreateViaLeadAPI`.

---

## 2. Server-Side Lead API Ingestion Contract

### 2.1. The Exact 8 Accepted Fields (`PublicCreateLeadRequest`)

Located in `internal/modules/leads/dto/public.go` and enforced in `internal/modules/leads/service/api_create.go`:

```go
type PublicCreateLeadRequest struct {
    Name           json.RawMessage `json:"name"`
    Email          json.RawMessage `json:"email"`
    Phone          json.RawMessage `json:"phone"`
    CompanyName    json.RawMessage `json:"company_name"`
    JobTitle       json.RawMessage `json:"job_title"`
    Source         json.RawMessage `json:"source"`
    EstimatedValue json.RawMessage `json:"estimated_value"`
    Notes          json.RawMessage `json:"notes"`
}
```

Every incoming field must be a **JSON string or null**. Any other type (e.g. sending a JSON number for `estimated_value` or a boolean) triggers `422 VALIDATION_ERROR`.

| Field | Type | Rules & Validation (`APICreateLeadInput.Validate()`) |
| :--- | :--- | :--- |
| **`name`** | `string` | **Required\*** (max 255 chars). \*Either `name` or `email` must be non-empty (`hasIdentity`). |
| **`email`** | `string` | **Required\*** (max 255 chars). Must be a valid email. Lowercased and trimmed for dedup. |
| **`phone`** | `string` | Max 50 chars. `leadPhoneWellFormed`: 7 to 15 digits once `[\s\-().]` are stripped, optional leading `+`. |
| **`company_name`** | `string` | Max 255 chars. Stored trimmed. |
| **`job_title`** | `string` | Max 150 chars. Stored trimmed. |
| **`source`** | `string` | Must match `allowedLeadSources`: `"Web form"`, `"Referral"`, `"Event"`, `"Cold outreach"`, `"Ad"`, `"LinkedIn"`, `"Import"`, `"API"`, `"Facebook"`. Missing or blank defaults to `"API"`. |
| **`estimated_value`** | `string` | Must be a decimal string (e.g. `"5000.00"`), max `999999999999.99`. Not a JSON number. |
| **`notes`** | `string` | Multi-line text (max 10,000 characters). Automatically creates a `note` timeline activity! |

### 2.2. Fields That Cannot Be Set via Public API
* **`owner_id`**: Always forced to the member who created the API key / webhook (`creatorID`).
* **`status_id`**: Always forced to the workspace's default active lead status.
* **`lead_score`**: Defaults to 0; computed solely by the backend scoring engine.
* **Enrichment fields** (`company_domain`, `industry`, `employees_size`, `expected_close_date`): Only accepted in internal app `POST /leads`, not in `PublicCreateLeadRequest`.

### 2.3. Server-Side Execution Flow

```
Incoming Webhook Request
   │
   ▼
[Webhook Gateway / Route Handler]
   │  Resolves endpoint_id & pulls stored mapping rules
   ▼
[Mapping & Transformation Engine]
   │  1. Extracts fields via JSONPath
   │  2. Concatenates first_name + last_name -> single "name"
   │  3. Normalizes "source" and formats "estimated_value"
   ▼
APICreateLeadInput (internal/modules/leads/service/api_create.go)
   │
   ▼
s.CreateViaLeadAPI(ctx, wsID, creatorID, in)
   ├── in.Validate() (Identity check: name or email required, phone regex)
   ├── prepareLeadWrite (Sets OwnerID = creatorID, resolves default StatusID)
   ├── s.createLead()
   │     ├── Duplicates Check: honors workspace "allow_duplicate_leads" setting
   │     ├── Records Cap: reserveLeadRecord charges quota before insert
   │     ├── DB INSERT INTO leads
   │     ├── Timeline entry 1: "Created via Lead API" (via: lead_api)
   │     └── Timeline entry 2: note activity with notes content (if provided)
   ▼
201 Created: { "id": "<lead-uuid>", "url": "https://app.nrtur.io/leads/<lead-uuid>" }
```

---

## 3. Separation of Concerns: API Keys vs. Webhook Endpoints

> [!NOTE]
> **API Keys and Endpoints are Completely Decoupled:**
> 1. **API Keys (Authentication & Identity):**
>    * Created manually by the user via the `+ New key` button in the existing **Lead API** card.
>    * Up to 10 active keys per workspace (`Default key`, `create Leads`, etc.).
>    * Full lifecycle: Rename, Regenerate, Revoke.
>    * The key's creator automatically becomes the Lead's owner in Nrtur.
> 2. **Webhook Endpoints (Routing & Mapping):**
>    * Created separately to define how an external payload is shaped (e.g. Shopify, Meta Lead Ads).
>    * **Endpoints DO NOT generate separate keys.**
>    * Any incoming request to a webhook endpoint MUST present one of the workspace's active API keys (`Authorization: Bearer <key>` or `?api_key=<key>`).
>    * Requests without a valid API key are immediately rejected with `401 UNAUTHORIZED`.

---

## 4. The 3 Ingestion Pathways Explained

| Pathway | Target Route | Sender Characteristics | How Mapping Works |
| :--- | :--- | :--- | :--- |
| **Pathway 1: Native Lead API** | `POST /api/public/v1/leads` | Standard forms, Zapier, Make | Sender already sends Nrtur's 8 exact fields with single `name`. |
| **Pathway 2: In-Payload Mapping** | `POST /api/public/v1/leads/dynamic` | Programmable clients, custom middleware | Caller sends payload + `_mapping` block in the request body. |
| **Pathway 3: Dynamic Webhook Endpoints** | `POST /api/public/v1/webhooks/:id` | **Shopify, Meta Lead Ads, Stripe** | Caller sends arbitrary payload. Nrtur loads pre-configured mapping rules from DB, combines `first_name + last_name` into `name`, and ingests. |

---

## 4. Frontend Integration (`nrtur-frontend`)

### 4.1. Page Placement
In `src/features/settings/components/SettingsIntegrationsPage.tsx`, the dynamic webhook card mounts right below the existing `LeadApiCard`:

```tsx
export function SettingsIntegrationsPage() {
  return (
    <div className="flex animate-in flex-col gap-8 duration-500 fade-in">
      <MailAccountsProvider>
        <EmailAccountsCard />
      </MailAccountsProvider>
      <LeadApiCard />
      {/* New: Dynamic Webhooks & Ingestion Card */}
      <LeadWebhookCard />
    </div>
  );
}
```

### 4.2. Review Files in `D:\Nature\Artifact\`
* **[`nrtur-lead-webhooks-ui.html`](file:///D:/Nature/Artifact/nrtur-lead-webhooks-ui.html)**: Interactive UI prototype strictly implementing Nrtur's exact 8-field Lead model, single `name` concatenation, and the 3 pathways.
* **[`index.html`](file:///D:/Nature/Artifact/index.html)**: General architectural concept simulator.
