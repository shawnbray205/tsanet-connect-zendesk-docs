# Inbound Setup (ZIS)

This page configures the **inbound path**: TSANet Connect pushes events to Zendesk in real time. When a partner accepts your case, posts a note, requests more information, or closes the collaboration, your Zendesk ticket updates automatically — no polling, no delays.

The entire inbound path is handled by **Zendesk Integration Services (ZIS)** — no server required.

---

## Step 1 — Create the ZIS Integration

A ZIS integration is the container that holds your inbound webhook and flows. Create it once.

```bash
curl -X POST "https://{subdomain}.zendesk.com/api/services/zis/integrations" \
  -u "{email}/token:{api_token}" \
  -H "Content-Type: application/json" \
  -d '{
    "integration": {
      "name": "tsanet_connect",
      "description": "TSANet Connect 3.0 collaboration event receiver"
    }
  }'
```

**Save the `name` value (`tsanet_connect`)** — you will reference it in every subsequent ZIS API call.

---

## Step 2 — Create the Inbound Webhook

This gives you a unique URL that TSANet will POST events to.

```bash
curl -X POST "https://{subdomain}.zendesk.com/api/services/zis/inbound_webhooks" \
  -u "{email}/token:{api_token}" \
  -H "Content-Type: application/json" \
  -d '{
    "inbound_webhook": {
      "name": "tsanet_events",
      "integration_name": "tsanet_connect",
      "description": "Receives collaboration lifecycle events from TSANet Connect 3.0"
    }
  }'
```

The response includes a `path` field — **this is your inbound webhook URL.** It looks like:

```
https://{subdomain}.zendesk.com/api/services/zis/inbound_webhooks/generic/tsanet_connect
```

**Give this URL to TSANet** (membership@tsanet.org) to register it as your webhook endpoint in the TSANet Connect integration settings. TSANet will configure their platform to POST collaboration events to this URL.

{% hint style="info" %}
Until TSANet confirms your webhook endpoint is registered, you can test the inbound URL manually using `curl` or Postman — send a POST with a sample event payload and verify the ZIS flow triggers correctly.
{% endhint %}

---

## Step 3 — Understand the TSANet Event Schema

TSANet Connect 3.0 webhook events follow this structure:

```json
{
  "eventType": "collaboration.status.changed",
  "token": "abc123xyz",
  "status": "ACCEPTED",
  "priority": "HIGH",
  "respondBy": "2026-04-01T14:00:00Z",
  "submitterCaseNumber": "ZD-98765",
  "receiverCaseNumber": "INC-44123",
  "partnerCompanyName": "NetApp",
  "engineerName": "Bob Engineer",
  "engineerEmail": "bob@netapp.com",
  "nextSteps": "Please run the diagnostic and attach the output log."
}
```

**For note events:**

```json
{
  "eventType": "collaboration.note.created",
  "token": "abc123xyz",
  "note": {
    "summary": "Diagnostic complete",
    "description": "Detected SCSI timeout at 14:23 UTC. Can you check HBA firmware version?",
    "priority": "HIGH",
    "visibility": "PARTNER_ONLY",
    "createdBy": "bob@netapp.com",
    "createdAt": "2026-04-01T15:30:00Z"
  }
}
```

**Event types you need to handle:**

| `eventType` | When It Fires |
|---|---|
| `collaboration.status.changed` | Case accepted, rejected, closed, or moved to INFORMATION |
| `collaboration.note.created` | Partner posts a case note |
| `collaboration.sla.breach` | Initial response SLA deadline has passed |
| `collaboration.information.requested` | Partner requests more info before accepting (sets status to INFORMATION) |

{% hint style="warning" %}
**Confirm the exact event schema with TSANet** before going live. The structure above reflects the Connect 3.0 API model — TSANet may use slightly different field names in the actual webhook payload. Request a sample payload when you register your endpoint.
{% endhint %}

---

## Step 4 — Create the ZIS Flow

The flow is the logic that runs when your inbound webhook receives an event. It routes events by type and calls the Zendesk Tickets API to update the right ticket.

{% hint style="info" %}
ZIS flows are defined in JSON and uploaded via the ZIS API. The flow below handles all four event types in a single definition. Upload it, then test each path individually.
{% endhint %}

### 4a — The Ticket Lookup Helper

Every action requires finding the Zendesk ticket that corresponds to a TSANet token. Zendesk Search API lets you query by custom field value:

```
GET /api/v2/search.json?query=type:ticket custom_field_{field_id}:{token}
```

Replace `{field_id}` with the numeric ID of your `tsanet_token` custom field.

### 4b — Upload the Flow

```bash
curl -X POST "https://{subdomain}.zendesk.com/api/services/zis/flows" \
  -u "{email}/token:{api_token}" \
  -H "Content-Type: application/json" \
  -d @tsanet-flow.json
```

**`tsanet-flow.json`** — save this file and fill in your `{field_id}` and `{subdomain}`:

```json
{
  "flow": {
    "name": "tsanet_event_router",
    "integration_name": "tsanet_connect",
    "event_source": "tsanet_events",
    "flow_definition": {
      "states": {

        "start": {
          "type": "action",
          "action": "switch",
          "parameters": {
            "field": "$.eventType",
            "cases": [
              { "value": "collaboration.status.changed",       "next": "handle_status_change" },
              { "value": "collaboration.note.created",         "next": "handle_note" },
              { "value": "collaboration.information.requested","next": "handle_information_request" },
              { "value": "collaboration.sla.breach",           "next": "handle_sla_breach" }
            ],
            "default": "end"
          }
        },

        "handle_status_change": {
          "type": "action",
          "action": "zendesk.find_ticket_by_custom_field",
          "parameters": {
            "field_id": "{field_id}",
            "field_value": "$.token",
            "subdomain": "{subdomain}"
          },
          "result_path": "$.ticket",
          "next": "update_status_tag"
        },

        "update_status_tag": {
          "type": "action",
          "action": "zendesk.update_ticket",
          "parameters": {
            "ticket_id": "$.ticket.id",
            "subdomain": "{subdomain}",
            "tags": {
              "add": ["tsanet_{{ $.status | downcase }}"],
              "remove": ["tsanet_open","tsanet_information","tsanet_accepted","tsanet_rejected","tsanet_closed"]
            },
            "custom_fields": [
              { "id": "{field_id_status}", "value": "$.status" },
              { "id": "{field_id_respond_by}", "value": "$.respondBy" },
              { "id": "{field_id_partner}", "value": "$.partnerCompanyName" }
            ]
          },
          "next": "add_status_comment"
        },

        "add_status_comment": {
          "type": "action",
          "action": "zendesk.add_comment",
          "parameters": {
            "ticket_id": "$.ticket.id",
            "subdomain": "{subdomain}",
            "body": "[TSANet] Status updated to {{ $.status }} by {{ $.partnerCompanyName }}{% if $.engineerName %}. Assigned engineer: {{ $.engineerName }} ({{ $.engineerEmail }}){% endif %}{% if $.nextSteps %}. Next steps: {{ $.nextSteps }}{% endif %}",
            "public": false
          },
          "next": "end"
        },

        "handle_note": {
          "type": "action",
          "action": "zendesk.find_ticket_by_custom_field",
          "parameters": {
            "field_id": "{field_id}",
            "field_value": "$.token",
            "subdomain": "{subdomain}"
          },
          "result_path": "$.ticket",
          "next": "add_note_comment"
        },

        "add_note_comment": {
          "type": "action",
          "action": "zendesk.add_comment",
          "parameters": {
            "ticket_id": "$.ticket.id",
            "subdomain": "{subdomain}",
            "body": "[TSANet — {{ $.note.visibility == 'PARTNER_ONLY' ? 'Partner Only' : 'Customer + Partner' }}] {{ $.note.summary }}\n\n{{ $.note.description }}\n\n— {{ $.note.createdBy }} ({{ $.note.priority }} priority)",
            "public": false
          },
          "next": "end"
        },

        "handle_information_request": {
          "type": "action",
          "action": "zendesk.find_ticket_by_custom_field",
          "parameters": {
            "field_id": "{field_id}",
            "field_value": "$.token",
            "subdomain": "{subdomain}"
          },
          "result_path": "$.ticket",
          "next": "update_information_tag"
        },

        "update_information_tag": {
          "type": "action",
          "action": "zendesk.update_ticket",
          "parameters": {
            "ticket_id": "$.ticket.id",
            "subdomain": "{subdomain}",
            "tags": {
              "add": ["tsanet_information", "tsanet_action_required"],
              "remove": ["tsanet_open"]
            }
          },
          "next": "add_information_comment"
        },

        "add_information_comment": {
          "type": "action",
          "action": "zendesk.add_comment",
          "parameters": {
            "ticket_id": "$.ticket.id",
            "subdomain": "{subdomain}",
            "body": "⚠️ ACTION REQUIRED — TSANet Collaboration\n\n{{ $.partnerCompanyName }} has requested additional information before accepting this collaboration case.\n\nResponse required by: {{ $.respondBy }}\nTSANet Case Token: {{ $.token }}\n\nTo respond, go to the TSANet Web App (connect.tsanet.org), find the case by token, and use the 'Provide Information' action. Do not close or reassign this Zendesk ticket until the TSANet collaboration is resolved.",
            "public": false
          },
          "next": "end"
        },

        "handle_sla_breach": {
          "type": "action",
          "action": "zendesk.find_ticket_by_custom_field",
          "parameters": {
            "field_id": "{field_id}",
            "field_value": "$.token",
            "subdomain": "{subdomain}"
          },
          "result_path": "$.ticket",
          "next": "add_sla_breach_comment"
        },

        "add_sla_breach_comment": {
          "type": "action",
          "action": "zendesk.add_comment",
          "parameters": {
            "ticket_id": "$.ticket.id",
            "subdomain": "{subdomain}",
            "body": "🚨 TSANet SLA BREACH — {{ $.partnerCompanyName }} has not responded within the SLA window for this {{ $.priority }} priority collaboration.\n\nToken: {{ $.token }}\nSLA Deadline: {{ $.respondBy }}\n\nRefer to the escalation instructions in the TSANet Web App for this partner.",
            "public": false
          },
          "next": "add_sla_tag"
        },

        "add_sla_tag": {
          "type": "action",
          "action": "zendesk.update_ticket",
          "parameters": {
            "ticket_id": "$.ticket.id",
            "subdomain": "{subdomain}",
            "tags": {
              "add": ["tsanet_sla_breach"]
            }
          },
          "next": "end"
        },

        "end": {
          "type": "success"
        }

      },
      "initial_state": "start"
    }
  }
}
```

### 4c — Fill in Your Values

Before uploading, replace every placeholder:

| Placeholder | Replace With |
|---|---|
| `{field_id}` | Numeric ID of your `tsanet_token` custom field |
| `{field_id_status}` | Numeric ID of your `tsanet_status` custom field |
| `{field_id_respond_by}` | Numeric ID of your `tsanet_respond_by` custom field |
| `{field_id_partner}` | Numeric ID of your `tsanet_partner` custom field |
| `{subdomain}` | Your Zendesk subdomain (e.g., `yourcompany`) |

---

## Step 5 — Create a Zendesk Trigger for INFORMATION Alerts

When a case moves to INFORMATION status, you want the assigned agent to get an active alert — not just an internal comment they might miss. Create a Zendesk Trigger that fires when the `tsanet_action_required` tag is added:

**In Admin Center → Objects and rules → Business rules → Triggers → Add trigger:**

| Setting | Value |
|---|---|
| Trigger name | TSANet Action Required — Information Requested |
| **Conditions** | Ticket tags — Contains at least one — `tsanet_action_required` |
| **AND** | Ticket updated — Yes |
| **Actions** | Assign to (the TSANet-handling agent or group) |
| **Actions** | Send email to (assignee) with subject: `Action Required: TSANet partner needs more information — {{ticket.title}}` |

{% hint style="warning" %}
The INFORMATION status is the single most common point of integration failure. An agent who misses this prompt leaves a collaboration case stalled. The trigger + comment combination ensures two notification paths — the comment in the ticket feed and an email alert to the assignee.
{% endhint %}

---

## Step 6 — Verify the ZIS Flow

Test the flow before registering your endpoint with TSANet. Send a simulated event payload directly to your inbound webhook URL:

```bash
# Simulate a status change event
curl -X POST \
  "https://{subdomain}.zendesk.com/api/services/zis/inbound_webhooks/generic/tsanet_connect" \
  -u "{email}/token:{api_token}" \
  -H "Content-Type: application/json" \
  -d '{
    "eventType": "collaboration.status.changed",
    "token": "TEST-TOKEN-001",
    "status": "ACCEPTED",
    "priority": "HIGH",
    "respondBy": "2026-04-01T14:00:00Z",
    "partnerCompanyName": "Test Partner Co",
    "engineerName": "Test Engineer",
    "engineerEmail": "test@partner.com",
    "nextSteps": "This is a test. Please verify the comment appears on your ticket."
  }'
```

{% hint style="info" %}
You need a Zendesk ticket that already has `TEST-TOKEN-001` in its `tsanet_token` custom field for the lookup to succeed. Create a test ticket, manually set the field, then run this command.
{% endhint %}

**Check:**
- A new internal comment appears on the ticket with TSANet status details
- The tag `tsanet_accepted` is added to the ticket
- The `tsanet_status` custom field shows `ACCEPTED`

Repeat with `eventType: "collaboration.information.requested"` and verify the `⚠️ ACTION REQUIRED` comment appears and the INFORMATION trigger fires.

---

## Zendesk Tags Reference

Your integration creates and manages these tags:

| Tag | Meaning |
|---|---|
| `tsanet_initiate` | Agent has requested a collaboration (triggers outbound worker) |
| `tsanet_open` | Collaboration submitted, awaiting partner response |
| `tsanet_information` | Partner has requested more information |
| `tsanet_action_required` | Agent must take action (used with INFORMATION trigger) |
| `tsanet_accepted` | Partner has accepted and assigned an engineer |
| `tsanet_rejected` | Partner has declined the collaboration |
| `tsanet_closed` | Collaboration is closed |
| `tsanet_sla_breach` | Partner response SLA has been breached |

Use these tags to build Zendesk Views, reports, and escalation automations tailored to your team's workflow.
