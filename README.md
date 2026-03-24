# Zendesk Connector

TSANet Connect does not yet ship a managed Zendesk package. This guide provides everything you need to build a production-ready integration yourself using Zendesk's native tooling — no custom server required.

The approach uses **two Zendesk-native surfaces** plus **one small serverless function** that you deploy once and never touch again:

| Component | What It Does | Technology |
|---|---|---|
| **ZIS Inbound Webhook** | Receives TSANet events and writes back to Zendesk tickets in real time | Zendesk Integration Services (native) |
| **ZIS Flow** | Routes TSANet events to the right Zendesk actions (tag updates, comments, alerts) | Zendesk Integration Services (native) |
| **Outbound Worker** | Handles the multi-step TSANet API chain when an agent initiates a collaboration | Cloudflare Workers (single JS file) |

{% hint style="info" %}
**Estimated integration time:** 2 days for a developer-assisted setup following this guide end-to-end. For the zero-code Email-to-Case approach, see [Email-to-Case Bridge](email-to-case.md) — you can be functional in under an hour.
{% endhint %}

---

## Architecture Overview

```
AGENT INITIATES COLLABORATION
─────────────────────────────────────────────────────────────────

  Zendesk Ticket                Cloudflare Worker           TSANet Connect API
  ┌─────────────┐               ┌───────────────────┐       ┌──────────────────┐
  │ Agent adds  │               │ 1. Validate sig   │       │                  │
  │ tag:        │──── webhook ──▶ 2. POST /v1/login  │──────▶│ /v1/login        │
  │ tsanet_     │               │ 3. GET partner    │       │ /v1/partners/... │
  │ initiate    │               │ 4. GET form       │       │ /v1/forms/...    │
  │             │               │ 5. POST case      │       │ /v1/collab-reqs  │
  │ tsanet_     │◀── PUT ticket─│ 6. Write token    │       │                  │
  │ token: abc… │               └───────────────────┘       └──────────────────┘


PARTNER RESPONDS / UPDATES ARRIVE
─────────────────────────────────────────────────────────────────

  TSANet Connect               ZIS Inbound Webhook          Zendesk Ticket
  ┌──────────────┐             ┌──────────────────────┐     ┌─────────────────┐
  │ Status       │             │ ZIS Flow:            │     │                 │
  │ changes,     │─── webhook ─▶ - parse event type   │     │ Tags updated    │
  │ notes added, │             │ - find ticket by     │────▶│ Comments added  │
  │ SLA breach   │             │   tsanet_token field │     │ ⚠️ INFORMATION  │
  │              │             │ - call Zendesk API   │     │   banner shown  │
  └──────────────┘             └──────────────────────┘     └─────────────────┘
```

---

## How the Token Links Everything

Every TSANet collaboration case has a unique `token` — a string that is the primary key for all API operations. When your outbound worker creates a case, it writes this token back to a custom field on the Zendesk ticket. Every subsequent inbound event from TSANet carries the same token, which ZIS uses to find the right Zendesk ticket and update it.

**Nothing works without the token being stored.** This is the most important implementation detail in this guide.

---

## Pages in This Section

* [Prerequisites](documentation/zendesk-connector/prerequisites.md) — what you need before starting
* [Inbound Setup (ZIS)](documentation/zendesk-connector/inbound-setup.md) — receive TSANet events in Zendesk natively
* [Outbound Setup (Worker)](documentation/zendesk-connector/outbound-setup.md) — submit collaboration cases from Zendesk
* [Field Mapping Reference](documentation/zendesk-connector/field-mapping.md) — Zendesk ↔ TSANet field mapping
* [Testing Checklist](documentation/zendesk-connector/testing-checklist.md) — validate your integration end-to-end
* [Email-to-Case Bridge](documentation/zendesk-connector/email-to-case.md) — zero-code option for getting started immediately
