# Testing Checklist

Run through every test below before switching to the Production TSANet environment. Tests are ordered to build on each other — complete them in sequence.

Use `testSubmission: true` (automatically set when `TSANET_ENV=BETA`) throughout. Test submissions do not trigger real SLA clocks or notify partner engineers.

---

## Environment Setup Verification

- [ ] Worker deployed to Cloudflare and accessible at its workers.dev URL
- [ ] All Wrangler secrets set: `TSANET_USERNAME`, `TSANET_PASSWORD`, `ZENDESK_SUBDOMAIN`, `ZENDESK_EMAIL_TOKEN`, `ZENDESK_WEBHOOK_SECRET`, `TSANET_TOKEN_FIELD_ID`, `TSANET_DEFAULT_DEPT_ID`
- [ ] `TSANET_ENV` = `BETA`
- [ ] All four Zendesk custom fields created: `tsanet_token`, `tsanet_status`, `tsanet_partner`, `tsanet_respond_by`
- [ ] ZIS integration and inbound webhook created
- [ ] ZIS flow uploaded and active
- [ ] Zendesk webhook created, pointing to your worker URL
- [ ] Zendesk trigger active: fires on `tsanet_initiate` tag, sends payload to the webhook

---

## Test 1 — TSANet Authentication

Confirm your Beta credentials work before anything else.

```bash
curl -X POST https://connect2.tsanet.net/v1/login \
  -H "Content-Type: application/json" \
  -d '{"username": "your_api_user", "password": "your_api_password"}'
```

**Expected:** HTTP 200 with `{"accessToken": "eyJ...", "tokenType": "Bearer", "expiresIn": 3600}`

- [ ] Login returns a valid JWT token
- [ ] Call `GET /v1/me` with the token and confirm your company name and ID are correct

---

## Test 2 — Partner Lookup and Form Retrieval

Confirm you can find your test partner and retrieve their form.

```bash
# Search for partner (use your actual Beta test partner name)
TOKEN="eyJ..."
curl -H "Authorization: Bearer $TOKEN" \
  "https://connect2.tsanet.net/v1/partners/tsanet"
```

**Expected:** Array with at least one result containing `companyId` and/or `departmentId`.

```bash
# Retrieve the process form
DEPT_ID=512  # Use the departmentId from the search result
curl -H "Authorization: Bearer $TOKEN" \
  "https://connect2.tsanet.net/v1/forms/department/$DEPT_ID"
```

**Expected:** Form object with `documentId` and `customFields` array. Check whether any fields have `required: true` — if so, your test submission must include them.

- [ ] Partner search returns at least one result
- [ ] Form retrieval returns a `documentId`
- [ ] Any required custom fields are identified

---

## Test 3 — Manual Case Submission

Submit a test collaboration case directly via the API, without involving Zendesk yet.

```bash
curl -X POST https://connect2.tsanet.net/v1/collaboration-requests \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "documentId": 512,
    "internalCaseNumber": "ZD-TEST-001",
    "problemSummary": "Integration test — please ignore",
    "problemDescription": "This is an automated test submission. No action required.",
    "priority": "LOW",
    "submitterContactDetails": {
      "name": "Test Agent",
      "email": "test@yourcompany.com",
      "phone": ""
    },
    "customFields": [],
    "testSubmission": true
  }'
```

**Expected:** HTTP 200 or 201 with a `token` field in the response.

- [ ] Response includes a `token` value (copy it — you need it for Tests 4 and 5)
- [ ] Case is visible in the TSANet Web App at [connect.tsanet.org](https://connect.tsanet.org)
- [ ] Case status is `OPEN`

---

## Test 4 — ZIS Inbound: Status Change

Simulate a TSANet status change event arriving at your ZIS webhook. First, create a Zendesk test ticket and manually set its `tsanet_token` custom field to the token you got in Test 3.

```bash
# Send a simulated ACCEPTED event to your ZIS inbound webhook
curl -X POST \
  "https://{subdomain}.zendesk.com/api/services/zis/inbound_webhooks/generic/tsanet_connect" \
  -u "{email}/token:{api_token}" \
  -H "Content-Type: application/json" \
  -d '{
    "eventType": "collaboration.status.changed",
    "token": "TOKEN-FROM-TEST-3",
    "status": "ACCEPTED",
    "priority": "LOW",
    "respondBy": "2099-12-31T23:59:59Z",
    "partnerCompanyName": "TSANet Test Partner",
    "engineerName": "Test Engineer",
    "engineerEmail": "test@partner.com",
    "nextSteps": "Integration test — verifying ZIS flow."
  }'
```

**Check the Zendesk test ticket:**

- [ ] Tag `tsanet_accepted` is added to the ticket
- [ ] Tag `tsanet_open` is removed
- [ ] `tsanet_status` custom field shows `ACCEPTED`
- [ ] `tsanet_partner` custom field shows `TSANet Test Partner`
- [ ] An internal comment appears with engineer name and next steps

---

## Test 5 — ZIS Inbound: Note Received

```bash
curl -X POST \
  "https://{subdomain}.zendesk.com/api/services/zis/inbound_webhooks/generic/tsanet_connect" \
  -u "{email}/token:{api_token}" \
  -H "Content-Type: application/json" \
  -d '{
    "eventType": "collaboration.note.created",
    "token": "TOKEN-FROM-TEST-3",
    "note": {
      "summary": "Initial findings",
      "description": "We reviewed the logs. The issue appears to be in the storage layer. Can you provide the HBA firmware version?",
      "priority": "MEDIUM",
      "visibility": "PARTNER_ONLY",
      "createdBy": "test@partner.com",
      "createdAt": "2026-04-01T12:00:00Z"
    }
  }'
```

**Check the Zendesk test ticket:**

- [ ] An internal comment appears prefixed with `[TSANet — Partner Only]`
- [ ] The comment body contains the note summary and description
- [ ] The comment is internal (not visible to end user)

---

## Test 6 — ZIS Inbound: INFORMATION Requested

This is the most critical test. The INFORMATION status is the most commonly missed integration path.

```bash
curl -X POST \
  "https://{subdomain}.zendesk.com/api/services/zis/inbound_webhooks/generic/tsanet_connect" \
  -u "{email}/token:{api_token}" \
  -H "Content-Type: application/json" \
  -d '{
    "eventType": "collaboration.information.requested",
    "token": "TOKEN-FROM-TEST-3",
    "status": "INFORMATION",
    "partnerCompanyName": "TSANet Test Partner",
    "respondBy": "2099-12-31T23:59:59Z"
  }'
```

**Check the Zendesk test ticket:**

- [ ] Tag `tsanet_information` is added
- [ ] Tag `tsanet_action_required` is added
- [ ] The `⚠️ ACTION REQUIRED` internal comment appears with clear instructions
- [ ] The INFORMATION Zendesk Trigger fires and sends an email alert to the assignee

{% hint style="danger" %}
If this test passes but the trigger email does not arrive, check the trigger conditions. The trigger must fire on ticket **update** (not just ticket creation) and the tag condition must match `tsanet_action_required`. This is the path agents most often miss — the alert must be unmissable.
{% endhint %}

---

## Test 7 — Outbound: Full End-to-End via Zendesk

Now test the full outbound flow using the real Zendesk trigger.

1. Create a new Zendesk test ticket (or use your existing test ticket)
2. Clear the `tsanet_token` field if it has a value from Test 4
3. Add the tag `tsanet_initiate` to the ticket and save

**Check within 5–10 seconds:**

- [ ] Zendesk trigger fires (visible in trigger log under Admin Center)
- [ ] Worker receives the payload and returns 200 (check Cloudflare Worker logs: `wrangler tail`)
- [ ] `tsanet_token` custom field is populated on the ticket
- [ ] Tag `tsanet_initiate` is removed; `tsanet_open` is added
- [ ] An internal comment appears confirming the case was created with the token value
- [ ] The collaboration case appears in the TSANet Web App with status `OPEN`

---

## Test 8 — SLA Breach Alert

```bash
curl -X POST \
  "https://{subdomain}.zendesk.com/api/services/zis/inbound_webhooks/generic/tsanet_connect" \
  -u "{email}/token:{api_token}" \
  -H "Content-Type: application/json" \
  -d '{
    "eventType": "collaboration.sla.breach",
    "token": "TOKEN-FROM-TEST-3",
    "priority": "HIGH",
    "partnerCompanyName": "TSANet Test Partner",
    "respondBy": "2026-03-01T10:00:00Z"
  }'
```

**Check the Zendesk test ticket:**

- [ ] Tag `tsanet_sla_breach` is added
- [ ] Internal comment appears with `🚨 TSANet SLA BREACH` alert text

---

## Test 9 — Error Handling

Test that the worker fails gracefully.

Set `TSANET_DEFAULT_DEPT_ID` temporarily to an invalid value (e.g., `0`), then tag a test ticket with `tsanet_initiate`.

- [ ] Worker returns an error response (check Cloudflare logs)
- [ ] Ticket receives an internal comment with `⚠️ TSANet integration error` text
- [ ] Tag `tsanet_error` is added to the ticket
- [ ] Tag `tsanet_initiate` is removed (so the agent can retry cleanly)

Restore the correct `TSANET_DEFAULT_DEPT_ID` after this test.

---

## Pre-Production Sign-Off

Before switching to Production credentials:

- [ ] All 9 tests above pass
- [ ] Cloudflare Worker logs reviewed — no unexpected errors during testing
- [ ] Zendesk trigger log reviewed — triggers firing correctly
- [ ] ZIS flow execution log reviewed — no failed flow runs
- [ ] A real agent (not the developer) has walked through the agent experience end-to-end on the test ticket
- [ ] TSANet account manager notified that you are ready to switch to Production
- [ ] Production TSANet credentials requested and received

**To go live:** update Wrangler secrets for `TSANET_USERNAME`, `TSANET_PASSWORD`, and `TSANET_ENV` (set to `PRODUCTION`), then run `npx wrangler deploy`.
