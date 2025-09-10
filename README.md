## n8n Email Lead Capture Workflow

### Overview
An automated, production-ready workflow that processes forwarded emails, extracts lead details, enriches company data, and writes normalized records into Supabase. It is optimized for reliability, cost-efficiency, and maintainability.

### Files in this repo
- `Email Lead Capture v1.json` — the n8n workflow export to import.

---

## Setup

### Prerequisites
- n8n (cloud or self-hosted)
- Supabase project (free tier is fine)
- A dedicated mailbox for forwarding (Gmail/Outlook)

### 1) Create the database schema in Supabase
Run in Supabase SQL Editor:

```sql
-- A table for company information
CREATE TABLE companies (
    id BIGINT PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    name TEXT,
    domain TEXT UNIQUE,
    description TEXT,
    logo_url TEXT
);

-- A table to store the captured sales leads
CREATE TABLE leads (
    id BIGINT PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    first_name TEXT,
    last_name TEXT,
    email TEXT UNIQUE NOT NULL,
    -- This will link a lead to a company
    company_id BIGINT REFERENCES companies(id)
);
```

### 2) Import the n8n workflow
1. Open n8n → Workflows → Import from File.
2. Select `Email Lead Capture v1.json`.
3. Review node credentials and environment variables.

### 3) Configure credentials
- IMAP Email Trigger: connect the dedicated inbox to monitor forwards.
- Supabase: URL and service role key or anon key with appropriate RLS disabled for this table scope.
- HTTP Request (Clearbit Autocomplete): no API key required.

### 4) Optional email hygiene
- Configure Post-Run Actions on the IMAP node to move processed emails into `Processed` (and failures to `Failed`).

---

## Workflow Architecture

1. IMAP Email Trigger — monitors the forward mailbox and emits email payloads.
2. Parse Email (Code) — deterministic parsing of the forwarded content to extract:
   - `first_name`, `last_name`, `email`
   - `domain`, `company_name`
   - `status`, `message` for debug
3. IF: Lead Parsed? — validates a proper email exists; stops early if not.
4. Supabase: Create Lead (INSERT) — uses unique email constraint to avoid duplicates.
5. HTTP Request: Clearbit Autocomplete — enriches company by name.
6. IF: Company Data Found? — gracefully handles empty/failed responses.
7. Supabase: Lookup Company (SELECT by domain).
8. IF: Company Exists? — choose existing or create new.
9. Supabase: Create Company (INSERT) — upsert pattern via domain uniqueness.
10. Resolve Company ID (Code) — normalize ID from SELECT vs INSERT outputs.
11. Supabase: Link Lead → Company (UPDATE on lead).

---

## Core Parsing Strategy (Code Node)

### Why a Code Node?
- **Deterministic & reliable**: Consistent regex handling across Gmail/Outlook and quoted threads.
- **Fast & cost-free**: No external AI calls, zero token spend.
- **Customizable**: Easy to extend patterns and edge-case logic.
- **Debuggable**: Transparent code and testable functions.

### Implementation highlights
Layered extraction with intelligent fallbacks:

1) Detect the From line (handles Gmail/Outlook variants):

```javascript
const fromRegexes = [
  /^From:\s*(.+?)\s*<([^<>\s]+@[^<>\s]+\.[^<>\s]+)>/im, // Name + email
  /^From:\s*<([^<>\s]+@[^<>\s]+\.[^<>\s]+)>/im,          // Email only in brackets
  /^From:\s*([^<>\s]+@[^<>\s]+\.[^<>\s]+)/im            // Plain email
];
```

2) Name intelligence when missing in headers:
- Infer from email local part: `john.doe@example.com → John Doe`.
- Title case normalization.
- Graceful degradation to partial data if needed.

3) Company detection for personal domains:

```javascript
const genericDomains = ["gmail.com", "outlook.com", "yahoo.com", "hotmail.com"];
if (genericDomains.includes(domain.toLowerCase())) {
  // Scan body for patterns like: "at **Acme Corporation**" or "at Acme"
}
```

4) Validation & control flow:
- Email is validated before database operations.
- IF node halts the workflow early if invalid.

---

## Clearbit Enrichment

- Endpoint: `https://autocomplete.clearbit.com/v1/companies/suggest?query=<company_name>`
- No key required. Returns array of `{ name, domain, logo }` objects.
- Fallback behavior: if no match, proceed with minimal company using derived domain.

---

## Database Upsert Logic

- Leads: INSERT with unique `email` constraint; ignore conflicts.
- Companies: SELECT-by-domain → IF → INSERT when missing.
- Link: UPDATE the created/selected `company_id` on the lead row.

---

## Test Cases

Use these by forwarding into the monitored mailbox:

### 1) Gmail Forward
```
--------- Forwarded message ---------

From: Jane Smith <jane.smith@widgets.co>

Date: Wed, May 22, 2024 at 11:30 AM

Subject: Question

To: <your-sales-email@example.com>

Could you send me your pricing information?
```

### 2) Outlook Forward
```
----Original Message----
From: Peter Jones <p.jones@innovate.tech>
Sent: Wednesday, May 22, 2024 9:00 AM
To: Your Name <your-sales-email@example.com>
Subject: Partnership opportunity

Hi, I think there could be a synergy between our companies. Let's talk.
```

### 3) Reply Thread (quoted with >)
```
---------- Forwarded message ---------
From: Maria Garcia <maria@summit-logistics.com>
Date: Tue, May 21, 2024 at 3:15 PM
Subject: Re: Following up
To: <your-sales-email@example.com>

Yes, I'm available to chat tomorrow at 2 PM.

> On Tue, May 21, 2024 at 1:00 PM, You wrote:
>
> Hi Maria, are you free for a call this week?
>
```

### 4) No-Name Lead
```
---------- Forwarded message ---------
From: <info@global-corp.net>
Date: Tue, May 21, 2024 at 4:00 PM
Subject: Info Request
To: <your-sales-email@example.com>

Please send product datasheets.
```

### 5) Personal Email with Company Mention in Body
```
---------- Forwarded message ---------
From: Michael Chen <m.chen88@gmail.com>
Date: Thu, May 23, 2024 at 1:20 PM

Hi there, I'm the VP of Ops at **Acme Corporation**, and I'm interested in your services.
```

---

## Error Handling & Operational Guidance

- Clearbit timeouts/errors: workflow continues with domain-only company record.
- Duplicate leads: handled via `email` unique constraint.
- Post-Run Actions: move processed emails to `Processed`, errors to `Failed`.
- Add optional retries/delays if processing at scale.

---

## Production Tips

- Add an email validation service for higher data quality.
- Standardize company naming across sources.
- Add notifications (webhook/Slack/Email) on parse failures.
- Create dashboards for capture metrics.

---

## How to Run Locally (n8n)

1) Import the workflow JSON.
2) Fill in credentials (IMAP inbox, Supabase URL/key).
3) Activate the workflow and forward the test emails.

---

## License
MIT (or choose your preferred license).


