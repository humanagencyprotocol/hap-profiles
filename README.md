# HAP Authority Profiles

Authorization templates for the [Human Agency Protocol](https://humanagencyprotocol.org). Each profile defines what an AI agent is allowed to do within a specific domain — the bounds a human sets, and the Gatekeeper enforces.

> **Version 0.4** — March 2026

---

## What Profiles Define

A profile is a complete authorization schema. It specifies:

- **Bounds schema** — the limits a human commits to (e.g., max amount, allowed currencies)
- **Context schema** — local parameters that stay encrypted (e.g., allowed services, environment)
- **Execution context** — what the Gatekeeper checks at runtime, including cumulative limits
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

### email — Email Authority

Governs sending, drafting, and reading email via Gmail.

| Bound | Type | Purpose |
|-------|------|---------|
| `recipient_max` | per-email | Maximum recipients |
| `send_daily_max` | cumulative | Daily send/draft limit |
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

### deploy — Deploy Authority

Governs which services an agent may deploy and to which environments.

| Bound | Type | Purpose |
|-------|------|---------|
| `deploy_daily_max` | cumulative | Daily deployment limit |

| Context | Type | Purpose |
|---------|------|---------|
| `allowed_services` | subset | Permitted services |
| `environment` | enum | sandbox, staging, production |

| Path | Default TTL |
|------|-------------|
| `deploy-sandbox` | 24h |
| `deploy-staging` | 8h |
| `deploy-production` | 2h |

---

### data — Data Authority

Governs accessing and modifying business data: queries, schema changes, exports.

| Bound | Type | Purpose |
|-------|------|---------|
| `access_level` | enum | read, write, admin |
| `data_scope` | enum | public, internal, pii |
| `scope` | enum | external (production) or internal (sandbox) |
| `row_limit_max` | per-query | Maximum rows returned |
| `query_count_daily_max` | cumulative | Daily query limit |
| `export_row_count_daily_max` | cumulative | Daily exported row limit |

| Path | Default TTL |
|------|-------------|
| `data-read` | 8h |
| `data-write` | 2h |
| `data-export` | 1h |

---

### provision — Infrastructure Authority

Governs creating, modifying, or destroying infrastructure: DNS, compute, storage, secrets.

| Bound | Type | Purpose |
|-------|------|---------|
| `resource_type` | enum | dns, compute, storage, secret |
| `action_type` | enum | create, modify, delete |
| `scope` | enum | external (production) or internal (dev/sandbox) |
| `blast_radius` | enum | single, service, global |
| `change_count_daily_max` | cumulative | Daily change limit |

| Path | Default TTL |
|------|-------------|
| `provision-internal` | 8h |
| `provision-external` | 2h |
| `provision-destructive` | 1h |

---

## How Profiles Work

A human creates an authorization by selecting a profile and execution path, setting the bounds, and answering the gate questions. Domain owners cryptographically attest to the bounds. The Gatekeeper then enforces those bounds on every tool call:

```
Human sets bounds          Agent requests execution        Gatekeeper checks
amount_max: 80 EUR    ->   amount: 5 EUR              ->  5 <= 80, approved
action_type: [charge]      action_type: "charge"          "charge" in [charge], approved
amount_daily_max: 500      amount_daily: 423 (from log)   423 + 5 <= 500, approved
```

If any bound is exceeded, the Gatekeeper blocks execution.

All profiles require the same six gates: bounds, problem, objective, tradeoff, commitment, and decision owner.

---

## Community Profiles

Anyone can create and publish profiles through the [HAP Service Provider](https://humanagencyprotocol.com). Published profiles are immutable and versioned. There is no approval process — trust decisions are local.

---

See [humanagencyprotocol.org](https://humanagencyprotocol.org) for the full protocol specification.
