# TSANet Connect — Zendesk Integration Guide

TSANet Connect does not yet ship a managed Zendesk package. This guide provides everything you need to build a production-ready integration yourself using Zendesk's native tooling — no custom server required.

The integration uses **three components** that work together:

| Component | What It Does | Technology |
|---|---|---|
| **ZAF Sidebar App** | Agents view and act on TSANet collaboration cases directly inside Zendesk tickets. Handles Accept, Reject, Request Info, Add Note, and Close. Polls for inbound cases and mirrors TSANet notes to the ticket thread. | Zendesk Apps Framework (private app, ZIP upload) |
| **ZIS Bearer Token** | Stores a live TSANet JWT inside Zendesk's Integration Services layer so that Zendesk-side automation can call the TSANet API without handling authentication itself. | Zendesk Integration Services + GitHub Actions |
| **GitHub Actions** | Two scheduled jobs: (1) refreshes the ZIS bearer token before it expires, (2) checks for SLA breaches and tags Zendesk tickets to fire email alerts — runs every 10 minutes, no browser required. | GitHub Actions (free tier sufficient) |

{% hint style="info" %}
**Estimated setup time:** 2–3 hours following the Quick Start guides end-to-end. No development environment or custom server required.
{% endhint %}

---

## Architecture Overview

```
AGENT ACTS ON A COLLABORATION (ZAF Sidebar)
──────────────────────────────────────────────────────────────────

  Zendesk Ticket Sidebar          ZAF App (iframe)           TSANet Connect API
  ┌──────────────────┐            ┌────────────────────┐     ┌──────────────────┐
  │ TSANet Connect   │            │ Polls TSANet every │     │                  │
  │ panel shows:     │            │ 5 min (background) │────▶│ /v1/collab-reqs  │
  │ • Case status    │            │                    │     │ /v1/notes        │
  │ • SLA countdown  │◀── proxy ──│ Agent clicks       │────▶│ /v1/approval     │
  │ • Notes feed     │            │ Accept / Reject /  │     │ /v1/rejection    │
  │ • Partner info   │            │ Add Note / Close   │     │ /v1/notes (POST) │
  └──────────────────┘            └────────────────────┘     └──────────────────┘


SERVER-SIDE AUTOMATION (GitHub Actions — no browser needed)
──────────────────────────────────────────────────────────────────

  GitHub Actions (:00 and :50)
  │
  ├── refresh-token job
  │     TSANet /v1/login → fresh JWT
  │     DELETE old ZIS bearer connection
  │     POST new ZIS bearer connection
  │
  └── sla-monitor job
        TSANet: GET OPEN cases
        For each past-deadline case:
          Zendesk: find ticket by TSANet token field
          Zendesk: POST tag tsanet_sla_breached
          → Zendesk trigger fires → emails ticket assignee
```

---

## How the Token Links Everything

Every TSANet collaboration case has a unique `token` — a string that is the primary key for all API operations. The ZAF app reads this token from a custom field on the Zendesk ticket, then uses it for every TSANet API call on that ticket.

When the ZAF app creates a new outbound collaboration, it writes the returned token back to the ticket. When GitHub Actions monitors SLA deadlines, it searches for tickets by token field value.

**Nothing works without the token being stored in the Zendesk ticket field.** This is the most important implementation detail.

---

## What the ZAF App Does

**On tickets with no TSANet case (compact mode):**
- Collapses to a slim 44px bar — unobtrusive on regular support tickets
- Shows a **+ New** button to start a new outbound collaboration

**On tickets linked to a TSANet case (full panel):**
- Displays all active collaborations with status and partner info
- **SLA countdown** — color-coded timer on unacknowledged (OPEN) cases only
- **Action buttons:** Accept, Reject, Request Info, Respond, Add Note, Close
- **Notes feed** — all TSANet case notes, rendered as clean plain text
- **Notes mirrored to ticket thread** — TSANet notes are posted as Zendesk internal comments so agents can see partner communication in the ticket timeline without opening the sidebar

**Background behavior (while any Zendesk tab is open):**
- Polls TSANet every 5 minutes for new inbound cases
- Detects SLA breaches and tags tickets to fire email alerts

{% hint style="warning" %}
**ZIS inbound webhooks are not supported.** TSANet sends webhook notifications without an `Authorization` header, which ZIS requires for inbound webhook flows. All inbound sync is handled by the ZAF background poller (browser-based) and the GitHub Actions `sla-monitor` job (server-side). Do not attempt to configure a ZIS inbound webhook flow.
{% endhint %}

---

## Known Limitations

| Limitation | Workaround |
|---|---|
| ZIS inbound webhooks blocked — TSANet sends no auth header | ZAF background poller (every 5 min) + GitHub Actions sla-monitor |
| ZIS scheduled polling not viable — ZIS flows cannot call ZIS management endpoints (circular OAuth scope) | GitHub Actions runs on cron, no Zendesk infra required |
| Zendesk Views API silently ignores custom field columns | Add custom field columns to views manually in Admin Center → Workspaces → Views |
| ZIS custom integrations do not appear in Admin Center UI | Verify via `GET /api/services/zis/integrations/tsanet_connect/connections` API call |
| ZAF app binary updates via API are broken | Upload updated ZIP manually via Admin Center → Zendesk Support Apps → Update |

---

## Guides in This Section

* [ZAF App Quick Start](documentation/ZAF_Quick_Start.md) — install and configure the sidebar app (~30 min)
* [ZIS & GitHub Actions Quick Start](documentation/ZIS_Quick_Start.md) — set up bearer token refresh and SLA monitoring (~45 min)
* [Plain English Implementation Guide](documentation/Zendesk_PlainEnglish_Implementation_Guide_v2.5.docx) — full narrative walkthrough with context and rationale (v2.5)
* [Claude Code Skill](documentation/SKILL_TSANet_Connect.md) — drop this into `~/.claude/skills/tsanet-connect/SKILL.md` to give Claude Code expert knowledge of this integration

---

## Current App Version

**ZAF App: v1.0.29** (May 2026)

| Version | Key Change |
|---|---|
| v1.0.29 | Manifest cleanup before first member distribution: cleared dev-instance Field ID defaults so new installs require members to enter their own field IDs (was previously pre-populated with TSANet's dev instance values, causing silent failures on member installs) |
| v1.0.28 | TSANet notes mirrored to Zendesk ticket thread as internal comments (`syncNotesToZendesk`) |
| v1.0.27 | Add Note modal split into Subject + Details fields; eliminates duplication in TSANet web app |
| v1.0.25 | Adaptive height — compact 44px bar on non-TSANet tickets, full panel on TSANet tickets |
| v1.0.24 | App tray icon (128×128 transparent PNG) |
| v1.0.22 | SLA respondBy synced to Zendesk date field; GitHub Actions sla-monitor + Zendesk trigger |
| v1.0.20 | Close button hidden on inbound cases (submitter-only TSANet API restriction) |
| v1.0.18 | HTML stripped from TSANet note content before display |
| v1.0.16 | Accept fix — `engineerEmail` required field added; uses TSANet API username for domain validation |
| v1.0.15 | All action buttons fixed — replaced `prompt()`/`confirm()` with custom inline modal |
