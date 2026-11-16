# n8n-clay-webhook-enrichment-credit-guard

**A pre-enrichment gatekeeper that sits between HubSpot and Clay tables.** It checks ICP fit, deduplicates domains that were already enriched, enforces a daily enrichment credit cap, and sends only net-new, qualified accounts to your Clay waterfalls.

Built by DH Robin · dhRobin Systems · Revenue infrastructure architecture

> **Safe by default.** The workflow ships with `SANDBOX_MODE=true` and `DRY_RUN=true`. In that state nothing leaves n8n: no HubSpot reads or writes, and no Clay rows. Every decision is still evaluated and audited against 13 pinned, fictional fixtures.

---

## Table of Contents

1. [Why this exists](#1-why-this-exists)
2. [Architecture](#2-architecture)
3. [Repository layout](#3-repository-layout)
4. [Quick start](#4-quick-start)
5. [Webhook setup](#5-webhook-setup)
6. [Root-domain normalisation & ICP filtering rules](#6-root-domain-normalisation--icp-filtering-rules)
7. [Daily credit budget cap & cooldown ledger](#7-daily-credit-budget-cap--cooldown-ledger)
8. [HubSpot property requirements](#8-hubspot-property-requirements)
9. [Clay table setup](#9-clay-table-setup)
10. [Environment variables](#10-environment-variables)
11. [Node configuration](#11-node-configuration)
12. [Testing stages](#12-testing-stages)
13. [Security model](#13-security-model)
14. [Troubleshooting](#14-troubleshooting)
15. [Known limitations](#15-known-limitations)
16. [Disclaimer & license](#16-disclaimer--license)

---

## 1. Why this exists

Clay waterfalls are powerful and they cost credits on every row. When HubSpot pushes every new company and contact straight into a Clay table, I usually see the same leaks:

- **Free-mail and disposable sign-ups** (`gmail.com`, `mailinator.com`) burn credits and return nothing useful.
- **Duplicate domains.** Ten contacts from one company trigger ten company lookups.
- **Re-enrichment.** Records enriched last month get enriched again because a property changed.
- **Out-of-ICP accounts.** Two-person studios, excluded industries and unsupported regions are enriched anyway.
- **No ceiling.** An import or a form-spam burst can drain a month of credits in an afternoon.

This workflow puts one deterministic gate in front of Clay. Every HubSpot object gets one of five outcomes, and each outcome is written back to HubSpot with a machine-readable reason code:

| Decision | HubSpot status | Meaning |
|---|---|---|
| `dispatch` | `queued` | Net-new, ICP-qualified, inside the daily budget. Sent to Clay. |
| `rejected` | `rejected` | Not enrichable: free/disposable/excluded domain, invalid domain, missing data. |
| `disqualified` | `disqualified` | Valid domain, but fails an ICP rule (employees, industry, country). |
| `skipped` | `skipped` | Valid and in-ICP, but not now: duplicate domain, cooldown, daily cap, lookup failure. |
| `ignored` | *(not written)* | Event not relevant (unwatched property change, duplicate event in the same payload). |

---

## 2. Architecture

![n8n Clay Webhook Enrichment Credit Guard Architecture](./workflow-overview.png)

```mermaid
flowchart TD
    A["HubSpot Webhook Trigger"] --> B["Parse & Authenticate Events"]
    B --> C{"Needs Record Fetch?"}
    C -- yes --> D["Fetch HubSpot Record<br/>GET /crm/v3/objects/:type/:id"]
    D --> E["Merge Fetched Record"]
    E --> F
    C -- no --> F["Gatekeeper<br/>root domain · free/disposable · ICP · record cooldown"]
    F --> G{"Needs Domain Lookup?"}
    G -- yes --> H["HubSpot Domain Cooldown Lookup<br/>POST /crm/v3/objects/companies/search"]
    H --> I["Apply Domain Lookup Result<br/>fails closed"]
    I --> J
    G -- no --> J["Credit Guard<br/>ledger dedupe · daily cap · credit reservation"]
    J --> K{"Route Decision"}
    K -- clay_live --> L["Dispatch to Clay Table<br/>HTTP POST, 3 retries"]
    L -- success --> M["Record Clay Dispatch"]
    L -- error --> N["Sanitize Clay Error<br/>& Refund Credits"]
    K -- clay_dry_run --> O["Simulate Clay Dispatch"]
    K -- skip_or_reject --> P
    M --> P["Build HubSpot Status Update"]
    N --> P
    O --> P
    P --> Q{"Write to HubSpot?"}
    Q -- live --> R["Update HubSpot Enrichment Status<br/>PATCH"]
    R -- success --> S["Confirm HubSpot Write"]
    R -- error --> T["Sanitize HubSpot Error"]
    Q -- no --> U["Audit Logger<br/>redacted JSON line + ring buffer"]
    S --> U
    T --> U
    X["Error Trigger"] --> Y["Sanitized Error Handler"]
```

### Order of checks

I ordered the checks from cheapest to most expensive, so an API call or a credit is spent only when every free check has passed:

1. **Authentication**: shared secret or HubSpot v3 signature. Fails closed.
2. **Payload shape**: object type and ID present, record fetched (live/shadow only).
3. **Root-domain normalisation**: invalid or missing domains are rejected.
4. **Domain blocklists**: free-mail, disposable, custom blocked and excluded domains.
5. **ICP fit**: employee range, industry allow/deny lists, country allowlist.
6. **Record history**: the record itself was enriched or dispatched inside the cooldown.
7. **CRM domain lookup**: any company on the same root domain was enriched or dispatched inside the cooldown (HubSpot Search API).
8. **Ledger dedupe**: the root domain was already reserved inside the cooldown (n8n static data).
9. **Daily credit cap**: estimated cost fits inside today's remaining budget.
10. **Reserve and dispatch**: credits and the ledger slot are reserved *before* the Clay call, and refunded if Clay fails.

---

## 3. Repository layout

```text
n8n-clay-webhook-enrichment-credit-guard/
├── README.md        # This guide
├── workflow.json    # Importable n8n workflow (26 nodes incl. 3 sticky notes, 13 pinned fixtures)
├── DISCLAIMER.md    # Liability limitation & open-source notice
└── LICENSE          # MIT License
```

---

## 4. Quick start

1. In n8n, go to **Workflows → Import from File** and choose `workflow.json`.
2. Leave every environment variable unset. The safe defaults apply: sandbox mode and dry run.
3. Open the workflow and click **Execute workflow**. The trigger uses its 13 pinned fictional fixtures.
4. Open the **Audit Logger** output. Each fixture has a `fixtureNote` with the expected result, and every record shows `decision`, `status` and `reasonCode`.
5. In **Workflow settings → Error Workflow**, select this workflow so the Error Trigger branch handles failures.
6. Follow the [testing stages](#12-testing-stages) before you connect a live HubSpot portal or Clay table.

### n8n platform prerequisites

| Setting | Why |
|---|---|
| `N8N_BLOCK_ENV_ACCESS_IN_NODE=false` | Code nodes and HTTP expressions read configuration from `$env`. Recent n8n versions block env access by default. If access stays blocked, every Code node falls back to the safe defaults (sandbox and dry run), so nothing goes live by accident. |
| `NODE_FUNCTION_ALLOW_BUILTIN=crypto` | Needed for constant-time secret comparison and HubSpot v3 signature checks. Without it, a slower fallback comparison is used and `hubspot_v3` mode fails closed. |
| Single worker or queue concurrency of 1 for this workflow | The ledger and budget live in workflow static data, which isn't transactional. See [section 7](#7-daily-credit-budget-cap--cooldown-ledger). |

The node versions used are Webhook v2, Code v2, IF v2.2, Switch v3.2 and HTTP Request v4.2. If your n8n instance is older, upgrade it before importing.

---

## 5. Webhook setup

The trigger accepts two payload shapes. Pick one.

### Option A: HubSpot Workflow "Send a webhook" action (recommended)

This option gives you HubSpot-side enrollment filters, so obvious junk never reaches n8n.

1. Create a **company-based** workflow and, if you want, a **contact-based** one.
2. Enrollment trigger example: *Record created* **and** `clay_enrichment_status` is unknown. For companies, also check *Company domain name is known*.
3. Add the action **Send a webhook**:
   - Method: `POST`
   - URL: `https://YOUR_N8N_HOST/webhook/hubspot-clay-credit-guard?objectType=company` (use `objectType=contact` in the contact workflow)
   - Authentication: send an API key **in a request header** named `x-webhook-secret`, with the value of `WEBHOOK_SHARED_SECRET`.
   - Body: either *include all properties*, or a customised body that includes at least `hs_object_id` plus `domain`/`website` (companies) or `email` (contacts).
4. Turn on re-enrollment only for the domain/email change you care about.

Both the legacy `{ "properties": { "email": { "value": "..." } } }` shape and flat key/value bodies are normalised. With `ALWAYS_FETCH_RECORD=true` (the default), the workflow fetches the fresh record from the CRM anyway, so the webhook body only needs the object ID.

### Option B: HubSpot Webhooks API (app subscriptions)

1. In your HubSpot app's webhook settings, set the target URL to `https://YOUR_N8N_HOST/webhook/hubspot-clay-credit-guard`.
2. Subscribe to `company.creation`, `company.propertyChange` (property `domain`), `contact.creation` and `contact.propertyChange` (property `email`).
3. Set `WEBHOOK_AUTH_MODE=hubspot_v3`, `HUBSPOT_APP_CLIENT_SECRET=YOUR_HUBSPOT_APP_CLIENT_SECRET` and `HUBSPOT_WEBHOOK_PUBLIC_URL` to the exact public URL HubSpot calls.
4. Turn on **Raw Body** in the trigger's options, so the signature is computed over the exact bytes HubSpot sent. The parser reads the raw body when it's present and falls back to re-serialised JSON when it isn't, which can cause signature mismatches.

Event arrays are split into one item per object. Events are deduplicated within one payload and capped at `MAX_EVENTS_PER_EXECUTION`. Property changes outside `WATCHED_PROPERTY_CHANGES` are ignored. That filter also stops the workflow's own `clay_*` writes from triggering it again.

### Response behaviour

The trigger replies `200` as soon as the request arrives (`responseMode: onReceived`). HubSpot stops retrying, and all filtering happens asynchronously. Rejected requests are logged by the Audit Logger instead of being returned to the caller, so an attacker learns nothing from the response.

---

## 6. Root-domain normalisation & ICP filtering rules

### Root-domain normalisation

| Input | Root domain |
|---|---|
| `https://www.app.brightforge-analytics.example/pricing?utm_source=x` | `brightforge-analytics.example` |
| `https://WWW.Shop.Example.co.uk/x` | `example.co.uk` |
| `jane@mail.acme-tools.com.au` | `acme-tools.com.au` |
| `acme.io:8443` | `acme.io` |
| `user:pass@evil.example.com/path` | `example.com` |
| `co.uk`, `localhost`, `192.168.0.1`, `n/a` | *rejected* (`invalid_root_domain`) |

The rules are:

1. Lowercase and trim. Take the part after `@` for emails.
2. Strip the protocol, credentials, path, query, fragment, port and trailing dots.
3. Every label must be a valid DNS label. The TLD must be alphabetic or punycode (`xn--`).
4. Reduce to the registrable domain. Two-label public suffixes (`co.uk`, `com.au`, `co.jp`, `com.br` and about 35 more) keep three labels. You can add more with `EXTRA_MULTI_PART_SUFFIXES`.
5. Source priority: companies use `domain`, then `website`. Contacts use `email`, then `hs_email_domain`.

### Domain blocklists

| Reason code | Source |
|---|---|
| `free_email_domain` | About 70 built-in consumer providers (Gmail, Outlook, Yahoo regional, iCloud, Proton, GMX, Yandex, QQ, Naver and others) |
| `disposable_email_domain` | About 30 built-in disposable providers (Mailinator, Guerrilla Mail, YOPmail, 10MinuteMail and others) |
| `blocked_domain` | `EXTRA_BLOCKED_DOMAINS`, comma-separated |
| `excluded_domain` | `EXCLUDED_DOMAINS`: your own domain, existing customers, partners, competitors |

### ICP rules

| Rule | Default | Reason code |
|---|---|---|
| Minimum employees | `ICP_MIN_EMPLOYEES=20` | `icp_employees_below_min` |
| Maximum employees | `ICP_MAX_EMPLOYEES=5000` | `icp_employees_above_max` |
| Industry allowlist | `ICP_INDUSTRIES=` (empty means any) | `icp_industry_not_allowed` |
| Industry denylist | `GOVERNMENT_ADMINISTRATION,NON_PROFIT_ORGANIZATION_MANAGEMENT,EDUCATION_MANAGEMENT` | `icp_industry_excluded` |
| Country allowlist | `United States,Canada,United Kingdom,Ireland,Australia,Germany,Netherlands` | `icp_country_not_allowed` |
| Unknown firmographics | `ICP_ALLOW_UNKNOWN_FIRMOGRAPHICS=true` | `icp_*_unknown` when set to `false` |

Matching details:

- **Employees**: companies use `numberofemployees` (a number). Contacts use `numemployees`, which is usually a range such as `50-100` or `1000+`. A range passes if it **overlaps** the ICP window.
- **Industry**: normalised to HubSpot's enum style, so `Computer Software` matches `COMPUTER_SOFTWARE`.
- **Country**: lowercased, with common aliases (`US`, `USA`, `UK`, `GB`, `CA`, `DE`, `NL` and others).
- **Unknown values**: enrichment is often how firmographics get filled in, so missing values **pass** by default. Set `ICP_ALLOW_UNKNOWN_FIRMOGRAPHICS=false` to require them.

---

## 7. Daily credit budget cap & cooldown ledger

### Cooldown (three layers)

| Layer | Where | Catches |
|---|---|---|
| Record history | The object's own `clay_last_enriched_at` / `clay_last_dispatched_at` | The same record coming back through the trigger |
| CRM domain lookup | HubSpot company search on `domain` + timestamps ≥ cutoff | Another company record on the same domain that was already enriched |
| Ledger | n8n workflow static data, keyed by root domain | Bursts that arrive before HubSpot has been updated, including several contacts from one company |

The cooldown window is `ENRICHMENT_COOLDOWN_DAYS` (default 90). With `DEDUPE_SCOPE=domain` (the default), a contact and a company on the same domain count as duplicates. Use `domain_and_object_type` if your contact table runs person-level enrichment that should still happen after a company has been enriched.

If the CRM lookup fails (non-200 or a network error), the item is skipped with `domain_lookup_failed`. When the gate can't confirm a domain is fresh, it doesn't spend credits.

### Daily credit budget

```text
cost        = CREDIT_COST_COMPANY (3) | CREDIT_COST_CONTACT (5)
day key     = today's date in BUDGET_TIMEZONE (default UTC)
if spent + cost > DAILY_CREDIT_BUDGET (150) → skipped: daily_credit_cap_reached
else        → spent += cost, ledger[rootDomain] = reserved, dispatch
on Clay error after retries → refund cost, delete ledger slot, status dispatch_failed
```

- Costs are **estimates per row**. Set them to the real average credit use of your Clay waterfall, and reconcile weekly against Clay's usage reporting.
- The budget rolls over automatically at midnight in `BUDGET_TIMEZONE`.
- A capped item is written back as `skipped` / `daily_credit_cap_reached`. If you want it retried tomorrow, have a HubSpot workflow re-enroll records where `clay_enrichment_skip_reason` is `daily_credit_cap_reached`.

### State modes

| Mode | Settings | Ledger and budget state |
|---|---|---|
| Sandbox | `SANDBOX_MODE=true` | Fresh and empty on every run, so the fixture results are always the same. Budget = `SANDBOX_DAILY_CREDIT_BUDGET` (10). |
| Shadow | `SANDBOX_MODE=false`, `DRY_RUN=true` | A read-only **copy** of the real static data. Decisions reflect real history, and nothing is saved. |
| Live | `SANDBOX_MODE=false`, `DRY_RUN=false` | The real workflow static data. Reservations persist. |

> **Important:** n8n saves workflow static data only for **production** executions (an active workflow triggered by a real webhook), not for manual test runs. It also isn't transactional across parallel executions. Run this workflow with a concurrency of 1, or replace the ledger with Redis or Postgres if you run several workers.

To reset the ledger immediately, add a temporary Schedule Trigger to a Code node that runs `delete $getWorkflowStaticData('global').clayCreditGuard`. Activate it for one tick, then remove it. In normal use, ledger entries expire on their own after the cooldown, and `LEDGER_MAX_ENTRIES` (20,000) caps memory use.

---

## 8. HubSpot property requirements

Create a property group named **Clay Enrichment** on **both Companies and Contacts**, then add these properties:

| Internal name | Label | Type / field type | Written by |
|---|---|---|---|
| `clay_enrichment_status` | Clay Enrichment Status | Enumeration / dropdown select | This workflow, and Clay write-back (`enriched`) |
| `clay_enrichment_skip_reason` | Clay Enrichment Skip Reason | String / single-line text | This workflow |
| `clay_enrichment_root_domain` | Clay Enrichment Root Domain | String / single-line text | This workflow |
| `clay_enrichment_checked_at` | Clay Enrichment Checked At | Datetime / date picker | This workflow |
| `clay_enrichment_run_id` | Clay Enrichment Run ID | String / single-line text | This workflow |
| `clay_last_dispatched_at` | Clay Last Dispatched At | Datetime / date picker | This workflow (on `queued`) |
| `clay_last_enriched_at` | Clay Last Enriched At | Datetime / date picker | **Clay write-back** |

`clay_enrichment_status` options (internal values): `queued`, `dispatch_failed`, `skipped`, `disqualified`, `rejected`, `enriched`.

Example request to create a property, with placeholders only:

```bash
curl -X POST "https://api.hubapi.com/crm/v3/properties/companies" \
  -H "Authorization: Bearer YOUR_HUBSPOT_PRIVATE_APP_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "clay_enrichment_status",
    "label": "Clay Enrichment Status",
    "groupName": "clay_enrichment",
    "type": "enumeration",
    "fieldType": "select",
    "options": [
      {"label": "Queued", "value": "queued", "displayOrder": 1},
      {"label": "Dispatch Failed", "value": "dispatch_failed", "displayOrder": 2},
      {"label": "Skipped", "value": "skipped", "displayOrder": 3},
      {"label": "Disqualified", "value": "disqualified", "displayOrder": 4},
      {"label": "Rejected", "value": "rejected", "displayOrder": 5},
      {"label": "Enriched", "value": "enriched", "displayOrder": 6}
    ]
  }'
```

Repeat the request for `/crm/v3/properties/contacts`. Create the `clay_enrichment` group first with `POST /crm/v3/properties/{objectType}/groups`.

### Private app scopes

| Scope | Used for |
|---|---|
| `crm.objects.companies.read` | Fetching companies and the domain cooldown search |
| `crm.objects.companies.write` | Writing status back to companies |
| `crm.objects.contacts.read` | Fetching contacts |
| `crm.objects.contacts.write` | Writing status back to contacts |
| `crm.schemas.companies.write`, `crm.schemas.contacts.write` | *Only* while you create the properties. Remove them afterwards. |

### Skip-reason codes written to HubSpot

`free_email_domain`, `disposable_email_domain`, `blocked_domain`, `excluded_domain`, `invalid_root_domain`, `missing_domain`, `missing_record_properties`, `icp_employees_below_min`, `icp_employees_above_max`, `icp_employee_count_unknown`, `icp_industry_excluded`, `icp_industry_not_allowed`, `icp_industry_unknown`, `icp_country_not_allowed`, `icp_country_unknown`, `recently_enriched_record`, `already_dispatched_record`, `domain_recently_enriched_in_crm`, `domain_lookup_failed`, `duplicate_domain_in_batch`, `duplicate_domain_ledger`, `daily_credit_cap_reached`, `clay_dispatch_failed`.

---

## 9. Clay table setup

1. Create one Clay table for companies and, if you need it, one for contacts.
2. Add a **Webhook** source to each table. Copy the URL into `CLAY_COMPANY_TABLE_WEBHOOK_URL` / `CLAY_CONTACT_TABLE_WEBHOOK_URL`. If you use one table for both, set only `CLAY_TABLE_WEBHOOK_URL`.
3. If the webhook source has an auth token, put it in `CLAY_WEBHOOK_AUTH_TOKEN`. It's sent as the `x-clay-webhook-auth` header.
4. Map these payload fields to columns:

```json
{
  "source": "n8n-clay-webhook-enrichment-credit-guard",
  "schema_version": 1,
  "run_id": "gate_…",
  "idempotency_key": "company:90000000001:2026-11-16",
  "object_type": "company",
  "hubspot_object_id": "90000000001",
  "root_domain": "brightforge-analytics.example",
  "company_name": "Brightforge Analytics",
  "website": "https://www.app.brightforge-analytics.example/pricing",
  "hints": { "employees": "240", "industry": "COMPUTER_SOFTWARE", "country": "united states" },
  "credit_guard": { "day_key": "2026-11-16", "estimated_credit_cost": 3 },
  "requested_at": "2026-11-16T01:25:00.000Z",
  "contact_email": "contacts only",
  "first_name": "contacts only, when CLAY_SEND_CONTACT_NAMES=true"
}
```

5. Build the enrichment waterfall. As the last step, use Clay's HubSpot integration to update the record by `hubspot_object_id`, setting `clay_enrichment_status=enriched` and `clay_last_enriched_at` to the current time. That write-back closes the cooldown loop.
6. Check Clay's current row limits for webhook sources, and archive or rotate tables before you reach them.

---

## 10. Environment variables

Every value below is a placeholder or a safe default. Never commit real values.

```bash
# ── Safety switches (both must be "false" before anything goes live) ──
SANDBOX_MODE=true
DRY_RUN=true

# ── Inbound authentication ──
WEBHOOK_AUTH_MODE=shared_secret            # shared_secret | hubspot_v3
WEBHOOK_SHARED_SECRET=YOUR_WEBHOOK_SHARED_SECRET
WEBHOOK_SECRET_HEADER=x-webhook-secret
HUBSPOT_APP_CLIENT_SECRET=YOUR_HUBSPOT_APP_CLIENT_SECRET   # hubspot_v3 only
HUBSPOT_WEBHOOK_PUBLIC_URL=https://YOUR_N8N_HOST/webhook/hubspot-clay-credit-guard

# ── HubSpot API ──
HUBSPOT_ACCESS_TOKEN=YOUR_HUBSPOT_PRIVATE_APP_TOKEN
HUBSPOT_API_BASE_URL=https://api.hubapi.com
ALWAYS_FETCH_RECORD=true
MAX_EVENTS_PER_EXECUTION=50
WATCHED_PROPERTY_CHANGES=domain,website,email
WRITE_SKIP_STATUS_TO_HUBSPOT=true

# ── Clay ──
CLAY_TABLE_WEBHOOK_URL=YOUR_CLAY_WEBHOOK_URL
CLAY_COMPANY_TABLE_WEBHOOK_URL=YOUR_CLAY_COMPANY_TABLE_WEBHOOK_URL
CLAY_CONTACT_TABLE_WEBHOOK_URL=YOUR_CLAY_CONTACT_TABLE_WEBHOOK_URL
CLAY_WEBHOOK_AUTH_TOKEN=YOUR_CLAY_WEBHOOK_AUTH_TOKEN
CLAY_SEND_CONTACT_NAMES=false

# ── ICP ──
ICP_MIN_EMPLOYEES=20
ICP_MAX_EMPLOYEES=5000
ICP_INDUSTRIES=
ICP_EXCLUDED_INDUSTRIES=GOVERNMENT_ADMINISTRATION,NON_PROFIT_ORGANIZATION_MANAGEMENT,EDUCATION_MANAGEMENT
ICP_COUNTRIES=United States,Canada,United Kingdom,Ireland,Australia,Germany,Netherlands
ICP_ALLOW_UNKNOWN_FIRMOGRAPHICS=true
EXTRA_BLOCKED_DOMAINS=
EXCLUDED_DOMAINS=your-own-domain.example
EXTRA_MULTI_PART_SUFFIXES=

# ── Credit guard & cooldown ──
ENRICHMENT_COOLDOWN_DAYS=90
DEDUPE_SCOPE=domain                         # domain | domain_and_object_type
DAILY_CREDIT_BUDGET=150
SANDBOX_DAILY_CREDIT_BUDGET=10
CREDIT_COST_COMPANY=3
CREDIT_COST_CONTACT=5
BUDGET_TIMEZONE=UTC
LEDGER_MAX_ENTRIES=20000

# ── Audit ──
AUDIT_LOG_TO_STDOUT=true
AUDIT_RING_BUFFER_SIZE=200

# ── n8n platform ──
N8N_BLOCK_ENV_ACCESS_IN_NODE=false
NODE_FUNCTION_ALLOW_BUILTIN=crypto
```

| Variable | Default | Notes |
|---|---|---|
| `SANDBOX_MODE` | `true` | Uses pinned/inline properties, never fetches from HubSpot, uses an empty ledger and the sandbox budget, and accepts only the fictional fixture secret when no real secret is set. |
| `DRY_RUN` | `true` | No Clay POST, no HubSpot PATCH, no persisted state. |
| Safety parsing | — | `SANDBOX_MODE` and `DRY_RUN` turn off only for exact `false`/`0`/`no`/`off`. Typos keep them on. |
| `WEBHOOK_SHARED_SECRET` | *(unset)* | If it's unset outside sandbox mode, **every request is rejected**. Use 32+ random characters. |
| `ALWAYS_FETCH_RECORD` | `true` | Re-reads the CRM record even when the webhook body has properties. |
| `WRITE_SKIP_STATUS_TO_HUBSPOT` | `true` | Set to `false` to write only `queued` / `dispatch_failed`, which saves HubSpot API calls. |
| `CLAY_SEND_CONTACT_NAMES` | `false` | Data minimisation. Contacts send only the email unless you turn this on. |
| `HUBSPOT_API_BASE_URL` | `https://api.hubapi.com` | Point it at a local mock server for integration tests. Only an administrator should be able to change it. |

---

## 11. Node configuration

| # | Node | Type | Mode / settings |
|---|---|---|---|
| 1 | HubSpot Webhook Trigger | Webhook v2 | `POST /webhook/hubspot-clay-credit-guard`, respond on receive, 13 pinned fixtures |
| 2 | Parse & Authenticate Events | Code | All items. Auth, payload shape detection, event splitting, in-payload dedupe |
| 3 | Needs Record Fetch? | IF | `$json.needsFetch` is true |
| 4 | Fetch HubSpot Record | HTTP Request | GET, full response, never error, 3 tries, continue on error |
| 5 | Merge Fetched Record | Code | Each item. CRM values override webhook values |
| 6 | Gatekeeper: Normalise, Filter & ICP | Code | Each item. Root domain, blocklists, ICP, record-level cooldown |
| 7 | Needs Domain Lookup? | IF | `$json.needsDomainLookup` is true |
| 8 | HubSpot Domain Cooldown Lookup | HTTP Request | POST companies/search, full response, never error |
| 9 | Apply Domain Lookup Result | Code | Each item. Fails closed |
| 10 | Credit Guard: Ledger & Daily Budget | Code | All items. Ledger, budget, reservation, Clay payload |
| 11 | Route Decision | Switch v3.2 | `clay_live` / `clay_dry_run` / fallback `skip_or_reject` |
| 12 | Dispatch to Clay Table | HTTP Request | POST, 15s timeout, 3 tries with 2s wait, **error output enabled** |
| 13 | Record Clay Dispatch | Code | Each item. Marks the ledger entry as dispatched |
| 14 | Sanitize Clay Error & Refund Credits | Code | Each item. Redacts the error, refunds credits, frees the ledger slot |
| 15 | Simulate Clay Dispatch (Dry Run) | Code | Each item. Masked payload preview, no network call |
| 16 | Build HubSpot Status Update | Code | Each item. Status mapping, `hubspotWrite` = `live` / `dry_run` / `not_applicable` / `disabled_for_skips` |
| 17 | Write to HubSpot? | IF | `$json.hubspotWrite` equals `live` |
| 18 | Update HubSpot Enrichment Status | HTTP Request | PATCH, 3 tries, **error output enabled** |
| 19 | Confirm HubSpot Write | Code | Each item. Restores context after the PATCH |
| 20 | Sanitize HubSpot Error | Code | Each item. Redacts the error and adds a hint for 400s |
| 21 | Audit Logger | Code | Each item. Redacted record to stdout plus a static-data ring buffer |
| 22 | Error Trigger | Error Trigger | Active when this workflow is its own Error Workflow |
| 23 | Sanitized Error Handler | Code | All items. No stack traces, secrets redacted |

**Credentials hardening (optional):** the HTTP nodes read tokens from `$env` so the repository stays credential-free. In production, I'd move `Authorization` and `x-clay-webhook-auth` into n8n **Header Auth** credentials. They're encrypted at rest and never appear in exported workflow JSON.

---

## 12. Testing stages

### Stage 1 · Sandbox (pinned fixtures, no network)

Settings: `SANDBOX_MODE=true`, `DRY_RUN=true` (the defaults). Click **Execute workflow** and compare the Audit Logger output with this table:

| # | Fixture (all fictional) | Expected decision | Reason code |
|---|---|---|---|
| 1 | Brightforge Analytics (subdomain + path URL) | dispatch → `dry_run` | — (3/10 credits) |
| 2 | maya.okafor@larkspur-robotics.example | dispatch → `dry_run` | — (8/10 credits) |
| 3 | sandbox.fixture.jordan@gmail.com | rejected | `free_email_domain` |
| 4 | fixture.temp@mailinator.com | rejected | `disposable_email_domain` |
| 5 | priya.raman@brightforge-analytics.example | skipped | `duplicate_domain_in_batch` |
| 6 | Tinyloop Studio (4 employees) | disqualified | `icp_employees_below_min` |
| 7 | Ventura Cacau Ltda (Brazil) | disqualified | `icp_country_not_allowed` |
| 8 | Civic Ledger Office (Government Administration) | disqualified | `icp_industry_excluded` |
| 9 | Northpine Freight (enriched 12 days ago) | skipped | `recently_enriched_record` |
| 10 | Quarry Peak Systems | skipped | `daily_credit_cap_reached` |
| 11 | theo.marsh@saltmarsh-health.example | skipped | `daily_credit_cap_reached` |
| 12 | Placeholder Holdings (website `n/a`) | rejected | `invalid_root_domain` |
| 13 | Spoofed Request Co (wrong secret) | rejected | `unauthorized` |

Fixture 9 uses `SANDBOX_DAYS_AGO:12`, which sandbox mode turns into a timestamp relative to the run time. The cooldown test therefore never goes stale.

### Stage 2 · Shadow (real reads, no writes, no credits)

Settings: `SANDBOX_MODE=false`, `DRY_RUN=true`, a real `WEBHOOK_SHARED_SECRET`, and a HubSpot token from a **developer test account or sandbox portal**.

- Unpin the trigger data, activate the workflow and point a HubSpot test workflow at the production URL.
- Create test companies and contacts. Confirm that the record fetch and domain lookup return `200` and that the decisions match your expectations.
- Nothing is written to HubSpot and no Clay rows are created. The ledger uses a read-only copy of real state.

### Stage 3 · Canary (live, tight cap)

Settings: `DRY_RUN=false`, `DAILY_CREDIT_BUDGET=15`, and a **test Clay table**.

- Confirm that rows arrive in Clay with the expected payload, and that HubSpot shows `queued` plus `clay_last_dispatched_at`.
- Break the Clay URL on purpose once. You should see `dispatch_failed` with a redacted reason and `creditsRefunded: true` in the audit record.
- Send two contacts from the same domain. The second should be `duplicate_domain_ledger`.

### Stage 4 · Production

Raise `DAILY_CREDIT_BUDGET` gradually, switch to the production Clay table, and review a week of audit records against Clay's credit usage before you calibrate `CREDIT_COST_*`.

---

## 13. Security model

| Control | Implementation |
|---|---|
| Fails closed on auth | Outside sandbox mode, a missing `WEBHOOK_SHARED_SECRET` rejects every request. `hubspot_v3` checks the HMAC-SHA256 signature and a 5-minute timestamp window. |
| Constant-time comparison | Both values are SHA-256 hashed, then compared with `crypto.timingSafeEqual`. |
| No secrets in item data | Clay URLs and tokens are resolved inside HTTP node expressions and never copied into item JSON. Request headers are dropped after authentication. |
| Redaction | Error and audit nodes redact bearer tokens, HubSpot `pat-…` tokens, Clay URLs, secret query parameters, keys that look like secrets, and the actual values of known secret env vars. Emails are masked as `m***@domain`. |
| No stack traces | The sanitized error handler forwards only the error name, a redacted message, the execution ID and the last node. |
| Fails closed on spend | Any uncertainty (a lookup failure, a fetch failure) skips the item instead of spending credits. |
| Data minimisation | Contacts send only the email to Clay by default. `clayPayload` is removed from item data after dispatch. |
| Safe defaults | Sandbox and dry run are on. Unparseable flag values fall back to the safe setting. Blocked env access also falls back to the safe setting. |
| Loop prevention | Property-change events outside `WATCHED_PROPERTY_CHANGES` are ignored, and records already `queued` inside the cooldown are skipped. |

**Residual risks to manage:**

- The **Webhook Trigger's own execution data contains the raw request headers**, including the shared secret. Restrict who can view executions, enable pruning (`EXECUTIONS_DATA_PRUNE=true`, `EXECUTIONS_DATA_MAX_AGE`), and consider `saveDataSuccessExecution: none` in production once the audit log is your system of record. `hubspot_v3` mode avoids sending a static secret at all.
- Clay receives contact email addresses. Confirm that you have a lawful basis and that your Clay data-processing terms cover this use.
- Rotate `WEBHOOK_SHARED_SECRET`, the HubSpot token and the Clay token on a schedule, and immediately if an execution export leaves your control.

---

## 14. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Every item is `rejected` / `unauthorized` | Secret missing or mismatched, or the header name differs | Check `WEBHOOK_SHARED_SECRET` and `WEBHOOK_SECRET_HEADER`, and the header configured in HubSpot. In the audit record, `reasonDetail` shows `shared_secret_not_configured` or `shared_secret_mismatch`. |
| `hubspot_v3_signature_mismatch` | Raw Body is off, or the URL differs | Turn on Raw Body on the trigger. Set `HUBSPOT_WEBHOOK_PUBLIC_URL` to the exact URL, including the query string. |
| Workflow seems stuck in sandbox mode | Env access is blocked | Set `N8N_BLOCK_ENV_ACCESS_IN_NODE=false` and restart n8n. |
| `missing_record_properties` | Sandbox mode with a Webhooks API payload (no inline properties) | Use the workflow-webhook fixtures, or move to shadow mode. |
| `record_fetch_failed` | 401 (token) or 429 (rate limit) | Check the private app scopes. Lower the HubSpot workflow's enrollment rate. |
| Lots of `domain_lookup_failed` | Search API rate limit, or missing `clay_*` properties | Create every property listed in section 8 and throttle the inbound rate. |
| `400` from the HubSpot PATCH | An enumeration option or property doesn't exist | Add the missing `clay_enrichment_status` options on **both** objects. |
| Duplicates still reach Clay | Manual test runs don't persist static data, or executions ran in parallel | Test dedupe across executions only in live/production mode, with concurrency 1. |
| Budget never resets | `BUDGET_TIMEZONE` is invalid | Use an IANA name such as `Asia/Dhaka` or `America/New_York`. Invalid values fall back to UTC. |
| `daily_credit_cap_reached` too early | `CREDIT_COST_*` set too high | Recalibrate against Clay usage. |
| Clay rows missing but status is `queued` | The Clay table hit a row limit, or a column mapping is wrong | Check the Clay webhook source history and table limits. |

---

## 15. Known limitations

- The public-suffix handling is a compact list, not the full Public Suffix List. Add suffixes through `EXTRA_MULTI_PART_SUFFIXES`.
- The CRM domain lookup matches the company `domain` property exactly. It doesn't search `hs_additional_domains`.
- Static data is a JSON blob saved after each execution. It isn't suitable for heavy parallel throughput. Swap in Redis or Postgres if you need that.
- Credit costs are estimates. Clay's real consumption depends on which waterfall steps run for each row.
- The free-mail and disposable lists are curated, not exhaustive. Extend them with `EXTRA_BLOCKED_DOMAINS`.

---

## 16. Disclaimer & license

### ⚠️ Disclaimer & Liability Limitation

Use at your own risk. The code, workflows, and scripts in this repository (`n8n-clay-webhook-enrichment-credit-guard`) are provided for educational and architectural demonstration purposes.

* **No Liability:** The author and dhRobin Systems accept no responsibility or liability for any direct, indirect, incidental, or consequential damages that result from using, misusing, or deploying these scripts. That includes unexpected Clay credit consumption, enrichment costs, or changes to CRM records.
* **No Warranty:** All software, n8n JSON blueprints, JavaScript modules and configuration examples are provided "AS IS", without warranty of any kind, express or implied. This includes, but is not limited to, warranties of merchantability, fitness for a particular purpose, and non-infringement.
* **Production Testing Required:** Always test thoroughly in an isolated staging environment, such as a HubSpot developer test account or sandbox and a dedicated Clay test table, with `SANDBOX_MODE=true` and `DRY_RUN=true` enabled, before you connect live HubSpot or Clay credentials. Treat the daily credit cap as a safeguard, not a guarantee, and check actual credit usage in Clay yourself.
* **Third-Party APIs & Compliance:** You alone are responsible for complying with the terms of service, rate limits, data privacy regulations (GDPR/CCPA), and security policies of HubSpot, Clay and any connected services. That includes confirming a lawful basis for sending contact and company data to third-party enrichment providers, honouring opt-outs and data-subject requests, and reviewing every automatically created or updated record.

The full disclaimer is also available in [DISCLAIMER.md](https://github.com/dhrobinsystems/n8n-clay-webhook-enrichment-credit-guard/blob/main/DISCLAIMER.md).

### License

Released under the MIT License. Copyright (c) 2026 DH Robin (dhRobin Systems). See [LICENSE](https://github.com/dhrobinsystems/n8n-clay-webhook-enrichment-credit-guard/blob/main/LICENSE).

---

Full text: DISCLAIMER.md · Licensed under the MIT License. License: MIT © 2026 DH Robin (dhRobin Systems) · Provided as-is, see the disclaimer.
