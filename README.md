# HAP Authority Profiles

Authorization templates for the [Human Agency Protocol](https://humanagencyprotocol.org). Each profile defines what an AI agent is allowed to do within a specific domain — the bounds a person sets. The Gatekeeper checks each call against them; the Authority Server enforces the cumulative ones (per day, week, month).

---

## What Profiles Define

A profile is a complete authorization schema. It specifies:

- **Bounds schema** — the limits a human commits to (e.g., max amount, allowed currencies)
- **Scope schema** — local constraints that partition the action space (e.g., allowed recipients, environment)
- **Execution context** — the values each call declares; the Gatekeeper checks them, and the Authority Server adds the running totals for cumulative limits
- **Execution paths** — governance tiers with required domain owners and TTLs
- **Gates** — the structured questions a human must answer before authorization

Profiles are referenced by ID (e.g., `charge@0.4`) and are immutable once published.

---

## Profiles

### charge — Charge Authority

Governs charging customers: payments, refunds, subscriptions.

| Bound | Type | Purpose |
|-------|------|---------|
| `amount_max` | per-transaction | Maximum monetary amount |
| `amount_daily_max` | cumulative | Daily charge cap |
| `amount_monthly_max` | cumulative | Monthly charge cap |
| `transaction_count_daily_max` | cumulative | Daily transaction limit |

| Context | Type | Purpose |
|---------|------|---------|
| `currency` | enum | Permitted currencies |
| `action_type` | enum | charge, refund, subscribe |

| Path | Default TTL |
|------|-------------|
| `charge-routine` | 24h |
| `charge-reviewed` | 4h |

---

### purchase — Purchase Authority

Governs spending company money: subscriptions, supplies, services, advertising.

| Bound | Type | Purpose |
|-------|------|---------|
| `spend_max` | per-transaction | Maximum spend amount |
| `spend_daily_max` | cumulative | Daily spend cap |
| `spend_monthly_max` | cumulative | Monthly spend cap |
| `transaction_count_daily_max` | cumulative | Daily transaction limit |

| Context | Type | Purpose |
|---------|------|---------|
| `currency` | enum | Permitted currencies |
| `category` | enum | subscription, supply, service, advertising |
| `allowed_vendors` | subset | Approved vendor names |

| Path | Default TTL |
|------|-------------|
| `purchase-routine` | 24h |
| `purchase-reviewed` | 4h |

---

### email — Email Authority

Governs sending, drafting, and reading email via Gmail.

| Bound | Type | Purpose |
|-------|------|---------|
| `recipient_max` | per-email | Maximum recipients |
| `send_daily_max` | cumulative | Daily send/draft limit |
| `send_weekly_max` | cumulative | Weekly send/draft limit, the last 7 days (from 0.11) |
| `read_max_age_days` | per-query | Max email age for search |
| `read_daily_max` | cumulative | Daily read limit |

| Context | Type | Purpose |
|---------|------|---------|
| `allowed_recipients` | subset | Permitted email addresses |
| `allowed_domains` | subset | Permitted recipient domains |

| Path | Default TTL |
|------|-------------|
| `email-draft` | 24h |
| `email-send` | 4h |
| `email-read` | 24h |

---

### customers — Customer Management

Governs CRM operations: contacts, activities, deals, tasks.

| Bound | Type | Purpose |
|-------|------|---------|
| `write_daily_max` | cumulative | Daily write operations |

| Context | Type | Purpose |
|---------|------|---------|
| `contact_type` | subset | customer, lead, partner, vendor |

| Path | Default TTL |
|------|-------------|
| `customers-read` | 24h |
| `customers-write` | 8h |
| `customers-delete` | 2h |

---

### schedule — Scheduling Authority

Governs calendar access: reading, drafting, and booking events.

| Bound | Type | Purpose |
|-------|------|---------|
| `booking_daily_max` | cumulative | Daily new bookings |
| `booking_duration_max` | per-event | Max event duration (minutes) |
| `lookahead_days_max` | per-event | Max days into the future |

| Context | Type | Purpose |
|---------|------|---------|
| `allowed_calendars` | subset | Permitted calendar names |
| `allowed_attendees` | subset | Permitted attendee emails |
| `allowed_domains` | subset | Permitted attendee domains |

| Path | Default TTL |
|------|-------------|
| `schedule-read` | 24h |
| `schedule-draft` | 24h |
| `schedule-book` | 4h |

---

### publish — Content Publishing

Governs posting public content to social media, blogs, and other platforms.

| Bound | Type | Purpose |
|-------|------|---------|
| `post_daily_max` | cumulative | Daily post limit |
| `post_weekly_max` | cumulative | Weekly post limit, the last 7 days (from 0.9) |
| `post_monthly_max` | cumulative | Monthly post limit |

| Context | Type | Purpose |
|---------|------|---------|
| `allowed_platforms` | subset | twitter, linkedin, instagram, blog, medium |
| `content_type` | enum | text, image, link, thread |
| `audience` | enum | public, followers, connections |

| Path | Default TTL |
|------|-------------|
| `publish-draft` | 24h |
| `publish-post` | 4h |

---

### records — Records Authority

Governs accessing and modifying personal structured data: queries, schema changes, exports. Renamed from `data` — for personal databases and spreadsheet replacements, not shared CRM data.

| Bound | Type | Purpose |
|-------|------|---------|
| `row_limit_max` | per-query | Maximum rows returned |
| `query_count_daily_max` | cumulative | Daily query limit |
| `export_row_count_daily_max` | cumulative | Daily exported row limit |

| Context | Type | Purpose |
|---------|------|---------|
| `access_level` | enum | read, write, admin |
| `data_scope` | enum | public, internal, pii |
| `scope` | enum | external (production), internal (sandbox) |

| Path | Default TTL |
|------|-------------|
| `records-read` | 8h |
| `records-write` | 2h |
| `records-export` | 1h |

---

### sales — Sales

Governs quote-to-order in an ERP: draft quotes, send quotes to customers (a binding offer), convert quotes to orders (reserves stock, commits a delivery date). No invoicing, delivery, payments or master data. Reference connector: `@humanagencyp/erp-mcp`, which re-derives the declared value and discount from the document and refuses a mismatch.

| Bound | Type | Purpose |
|-------|------|---------|
| `read_access` | enum | Look up items, stock, prices, customers, quotes, orders |
| `value_max` | per-transaction | Maximum net value of one quote or order |
| `discount_max` | per-transaction | Maximum discount against list price (percent) |
| `order_value_daily_max` | cumulative | Daily order value cap (orders only) |
| `quote_daily_max` | cumulative | Draft quotes per day |
| `send_daily_max` | cumulative | Quotes sent per day |
| `order_daily_max` | cumulative | Orders per day |

| Context | Type | Purpose |
|---------|------|---------|
| `currency` | enum | Permitted document currency |

---

## How Profiles Work

A person picks a profile, sets the bounds and answers the questions, and signs a **mandate**. On every call the Gatekeeper checks the call against the bounds; for a consequential action it asks the Authority Server for a ticket, and the Authority Server checks the cumulative limits before signing it:

```
Mandate bound              Call declares                 Checked by
amount_max: 80 EUR    ->   amount: 5 EUR            ->   Gatekeeper: 5 <= 80
action_type: [charge]      action_type: "charge"         Gatekeeper: allowed
amount_daily_max: 500      amount: 5 EUR                 Authority Server: 423 so far + 5 <= 500, ticket signed
```

If any bound is exceeded, nothing runs.

Every profile asks the same questions before a mandate is signed: the bounds, the intent, the commitment and the Mandate Owner.

---

## Community Profiles

Anyone can write a profile and propose it to this repository. Published profiles are immutable and versioned; whether to trust one is decided locally.

---

See [humanagencyprotocol.org](https://humanagencyprotocol.org) for the full protocol specification.
