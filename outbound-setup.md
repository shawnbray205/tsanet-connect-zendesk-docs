# outbound setup

This page configures the **outbound path**: an agent adds a tag to a Zendesk ticket, which triggers a Cloudflare Worker to chain the necessary TSANet API calls and write the returned collaboration token back to the ticket.

The worker is a single JavaScript file — approximately 150 lines — deployed to Cloudflare's free tier. It handles token caching, authentication, form fetching, case submission, and the write-back to Zendesk.

***

## How It Works

```
1. Agent adds tag `tsanet_initiate` to a Zendesk ticket
2. Zendesk Trigger fires → sends ticket payload to your Worker URL
3. Worker validates the Zendesk webhook signature
4. Worker calls POST /v1/login → gets JWT (cached for 50 minutes)
5. Worker calls GET /v1/forms/department/{departmentId} → gets process form
6. Worker maps ticket fields to TSANet collaboration request fields
7. Worker calls POST /v1/collaboration-requests → gets token
8. Worker calls PUT /api/v2/tickets/{id} → writes token to tsanet_token field
9. Worker calls PUT /api/v2/tickets/{id} → adds tag tsanet_open, removes tsanet_initiate
```

Steps 4–9 happen in about 1–2 seconds. The agent sees the ticket update almost immediately.

***

{% stepper %}
{% step %}
### Create the Worker Project

```bash
# Create the project
mkdir tsanet-zendesk-worker && cd tsanet-zendesk-worker
npm init -y
npm install -D wrangler

# Scaffold the wrangler config
npx wrangler init --no-delegate-c3
```

Replace the generated `wrangler.toml` with:

```toml
name = "tsanet-zendesk-worker"
main = "src/index.js"
compatibility_date = "2026-01-01"

[vars]
TSANET_ENV = "BETA"

# Secrets (set via `wrangler secret put`, not here)
# TSANET_USERNAME
# TSANET_PASSWORD
# ZENDESK_SUBDOMAIN
# ZENDESK_EMAIL_TOKEN    (format: "email@company.com/token:api_token_value")
# ZENDESK_WEBHOOK_SECRET
# TSANET_TOKEN_FIELD_ID  (numeric ID of tsanet_token custom field)
# TSANET_DEFAULT_DEPT_ID (default partner department ID — override per ticket if needed)
```
{% endstep %}

{% step %}
### Write the Worker

Create `src/index.js`:

```javascript
/**
 * TSANet Connect → Zendesk Outbound Worker
 * Cloudflare Workers (free tier)
 *
 * Triggered by a Zendesk webhook when an agent adds the
 * tag `tsanet_initiate` to a ticket. Chains the TSANet API
 * calls and writes the collaboration token back to the ticket.
 */

// ── TSANet API ───────────────────────────────────────────────────────────────

const TSANET_BASE = {
  BETA:       'https://connect2.tsanet.net/v1',
  PRODUCTION: 'https://connect2.tsanet.org/v1',
};

// Simple in-memory token cache. Cloudflare Workers are long-lived per isolate,
// so this cache survives across requests within the same instance.
let cachedToken = null;
let tokenExpiresAt = 0;

async function getTsanetToken(env) {
  const now = Date.now();
  if (cachedToken && now < tokenExpiresAt) return cachedToken;

  const base = TSANET_BASE[env.TSANET_ENV] ?? TSANET_BASE.BETA;
  const res = await fetch(`${base}/login`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      username: env.TSANET_USERNAME,
      password: env.TSANET_PASSWORD,
    }),
  });

  if (!res.ok) throw new Error(`TSANet login failed: ${res.status}`);

  const data = await res.json();
  cachedToken = data.accessToken;
  // Cache for 50 minutes (tokens typically expire at 60 minutes)
  tokenExpiresAt = now + 50 * 60 * 1000;
  return cachedToken;
}

async function tsanetGet(path, env) {
  const base = TSANET_BASE[env.TSANET_ENV] ?? TSANET_BASE.BETA;
  const token = await getTsanetToken(env);
  const res = await fetch(`${base}${path}`, {
    headers: { Authorization: `Bearer ${token}` },
  });
  if (!res.ok) throw new Error(`TSANet GET ${path} failed: ${res.status}`);
  return res.json();
}

async function tsanetPost(path, body, env) {
  const base = TSANET_BASE[env.TSANET_ENV] ?? TSANET_BASE.BETA;
  const token = await getTsanetToken(env);
  const res = await fetch(`${base}${path}`, {
    method: 'POST',
    headers: {
      Authorization: `Bearer ${token}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify(body),
  });
  if (!res.ok) {
    const err = await res.text();
    throw new Error(`TSANet POST ${path} failed ${res.status}: ${err}`);
  }
  return res.json();
}

// ── Zendesk API ──────────────────────────────────────────────────────────────

async function zendeskUpdateTicket(ticketId, updates, env) {
  const url = `https://${env.ZENDESK_SUBDOMAIN}.zendesk.com/api/v2/tickets/${ticketId}.json`;
  const res = await fetch(url, {
    method: 'PUT',
    headers: {
      Authorization: `Basic ${btoa(env.ZENDESK_EMAIL_TOKEN)}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({ ticket: updates }),
  });
  if (!res.ok) {
    const err = await res.text();
    throw new Error(`Zendesk ticket update failed ${res.status}: ${err}`);
  }
  return res.json();
}

// ── Priority Mapping ─────────────────────────────────────────────────────────

function mapPriority(zdPriority) {
  const map = { urgent: 'HIGH', high: 'HIGH', normal: 'MEDIUM', low: 'LOW' };
  return map[zdPriority] ?? 'MEDIUM';
}

// ── Webhook Signature Verification ───────────────────────────────────────────

async function verifyZendeskSignature(request, secret) {
  const signature = request.headers.get('x-zendesk-webhook-signature');
  const timestamp  = request.headers.get('x-zendesk-webhook-signature-timestamp');
  if (!signature || !timestamp) return false;

  const body = await request.clone().text();
  const message = timestamp + body;
  const key = await crypto.subtle.importKey(
    'raw', new TextEncoder().encode(secret),
    { name: 'HMAC', hash: 'SHA-256' }, false, ['sign']
  );
  const sig = await crypto.subtle.sign('HMAC', key, new TextEncoder().encode(message));
  const expected = btoa(String.fromCharCode(...new Uint8Array(sig)));
  return signature === expected;
}

// ── Main Handler ─────────────────────────────────────────────────────────────

export default {
  async fetch(request, env) {

    // Only accept POST
    if (request.method !== 'POST') {
      return new Response('Method not allowed', { status: 405 });
    }

    // Verify Zendesk webhook signature
    const valid = await verifyZendeskSignature(request, env.ZENDESK_WEBHOOK_SECRET);
    if (!valid) {
      console.error('Invalid webhook signature');
      return new Response('Unauthorized', { status: 401 });
    }

    let payload;
    try {
      payload = await request.json();
    } catch {
      return new Response('Invalid JSON', { status: 400 });
    }

    const ticket = payload.ticket;
    if (!ticket) return new Response('No ticket in payload', { status: 400 });

    const ticketId = ticket.id;

    try {
      // ── 1. Get the TSANet process form ───────────────────────────────────
      // departmentId can be overridden per-ticket using a custom field;
      // fall back to the environment default.
      const departmentId =
        ticket.custom_fields?.find(f => f.id?.toString() === env.TSANET_DEPT_FIELD_ID)?.value
        ?? env.TSANET_DEFAULT_DEPT_ID;

      if (!departmentId) {
        throw new Error('No TSANet department ID found. Set TSANET_DEFAULT_DEPT_ID or add a per-ticket field.');
      }

      const form = await tsanetGet(`/forms/department/${departmentId}`, env);

      // ── 2. Build the required custom fields from the form definition ─────
      // Include any required fields from the form. For required fields your
      // integration cannot auto-populate, the agent must set them via a
      // Zendesk custom field before tagging tsanet_initiate.
      const customFields = (form.customFields ?? [])
        .filter(f => f.required)
        .map(f => ({
          fieldId: f.fieldId,
          // Attempt to pull value from a matching Zendesk custom field.
          // Zendesk custom fields can be named to mirror TSANet field labels.
          value: ticket.custom_fields?.find(
            zf => zf.id?.toString() === f.fieldId?.toString()
          )?.value ?? '',
        }));

      // ── 3. Build and submit the collaboration request ────────────────────
      const collaborationRequest = {
        documentId: form.documentId,
        internalCaseNumber: String(ticket.id),
        problemSummary: ticket.subject ?? 'No subject',
        problemDescription: ticket.description ?? '',
        priority: mapPriority(ticket.priority),
        submitterContactDetails: {
          name:  ticket.requester?.name  ?? '',
          email: ticket.requester?.email ?? '',
          phone: ticket.requester?.phone ?? '',
        },
        customFields,
        testSubmission: env.TSANET_ENV === 'BETA', // Auto-test in Beta; real in Production
      };

      const result = await tsanetPost('/collaboration-requests', collaborationRequest, env);
      const tsanetToken = result.token;

      if (!tsanetToken) throw new Error('TSANet response did not include a token');

      // ── 4. Write the token back to the Zendesk ticket ───────────────────
      await zendeskUpdateTicket(ticketId, {
        custom_fields: [
          { id: parseInt(env.TSANET_TOKEN_FIELD_ID, 10), value: tsanetToken },
        ],
        tags: [...(ticket.tags ?? []).filter(t => t !== 'tsanet_initiate'), 'tsanet_open'],
        comment: {
          body: `TSANet collaboration case created.\n\nPartner: ${form.partnerName ?? `Department ${departmentId}`}\nPriority: ${collaborationRequest.priority}\nToken: ${tsanetToken}\n\nYou will receive updates here as the partner responds.`,
          public: false,
        },
      }, env);

      console.log(`Collaboration created: token=${tsanetToken} ticket=${ticketId}`);
      return new Response(JSON.stringify({ success: true, token: tsanetToken }), {
        headers: { 'Content-Type': 'application/json' },
      });

    } catch (err) {
      console.error(`Error processing ticket ${ticketId}:`, err.message);

      // Write a failure comment to the ticket so the agent knows something went wrong
      try {
        await zendeskUpdateTicket(ticketId, {
          comment: {
            body: `⚠️ TSANet integration error: Failed to create collaboration case.\n\nError: ${err.message}\n\nPlease try again or use the TSANet Web App (connect.tsanet.org) to submit manually. Remove and re-add the tsanet_initiate tag to retry.`,
            public: false,
          },
          tags: [...(ticket.tags ?? []).filter(t => t !== 'tsanet_initiate'), 'tsanet_error'],
        }, env);
      } catch (commentErr) {
        console.error('Failed to write error comment:', commentErr.message);
      }

      return new Response(JSON.stringify({ success: false, error: err.message }), {
        status: 500,
        headers: { 'Content-Type': 'application/json' },
      });
    }
  },
};
```
{% endstep %}

{% step %}
### Set Secrets

Never put credentials in `wrangler.toml`. Use Wrangler's secrets management:

```bash
# TSANet credentials
wrangler secret put TSANET_USERNAME
# Enter: your TSANet API username when prompted

wrangler secret put TSANET_PASSWORD
# Enter: your TSANet API password when prompted

# Zendesk credentials
wrangler secret put ZENDESK_SUBDOMAIN
# Enter: yourcompany (just the subdomain, no .zendesk.com)

wrangler secret put ZENDESK_EMAIL_TOKEN
# Enter: youremail@company.com/token:your_api_token_value
# (the Basic Auth credential string — will be base64-encoded by the worker)

wrangler secret put ZENDESK_WEBHOOK_SECRET
# Enter: the signing secret from your Zendesk webhook (see Step 4)

wrangler secret put TSANET_TOKEN_FIELD_ID
# Enter: the numeric ID of your tsanet_token custom field (e.g., 360012345678)

wrangler secret put TSANET_DEFAULT_DEPT_ID
# Enter: the TSANet departmentId for your primary partner (e.g., 512)
```
{% endstep %}

{% step %}
### Deploy the Worker

```bash
# Deploy to Cloudflare
npx wrangler deploy

# The output will show your worker URL:
# https://tsanet-zendesk-worker.{your-account}.workers.dev
```

Note the worker URL — you will need it in Step 5.
{% endstep %}

{% step %}
### Create the Zendesk Webhook

In **Admin Center → Apps and integrations → Webhooks → Webhooks → Create webhook:**

| Setting        | Value                                            |
| -------------- | ------------------------------------------------ |
| Name           | TSANet Outbound                                  |
| Endpoint URL   | Your Cloudflare Worker URL                       |
| Request method | POST                                             |
| Request format | JSON                                             |
| Authentication | None (the worker validates the signature itself) |

After creating the webhook, click **More options → Signing secret** and copy the generated signing secret. Set it as `ZENDESK_WEBHOOK_SECRET` using `wrangler secret put` (you may need to redeploy after updating the secret).
{% endstep %}

{% step %}
### Create the Zendesk Trigger

In **Admin Center → Objects and rules → Business rules → Triggers → Add trigger:**

**Trigger name:** `TSANet — Initiate Collaboration`

**Conditions (ALL):**

* Tags — Contains at least one of — `tsanet_initiate`
* Tag — Does not contain — `tsanet_open` _(prevents duplicate submissions)_

**Actions:**

* Notify active webhook — `TSANet Outbound`

**Webhook body** (paste this JSON exactly):

```json
{
  "ticket": {
    "id": "{{ticket.id}}",
    "subject": "{{ticket.title}}",
    "description": "{{ticket.description}}",
    "priority": "{{ticket.priority}}",
    "tags": "{{ticket.tags}}",
    "requester": {
      "name": "{{ticket.requester.name}}",
      "email": "{{ticket.requester.email}}",
      "phone": "{{ticket.requester.phone}}"
    },
    "custom_fields": {{ticket.custom_fields | to_json}}
  }
}
```

{% hint style="warning" %}
The `{{ticket.custom_fields | to_json}}` placeholder requires Zendesk's Liquid templating support. If your Zendesk plan does not support this syntax, replace it with individual field placeholders for each custom field you need. Verify the payload in the webhook test tool before going live.
{% endhint %}
{% endstep %}

{% step %}
### Test the Outbound Flow

Before tagging a real ticket:

```bash
# Test the worker directly (bypasses signature check for local testing)
# First, in wrangler.toml temporarily add:
# [dev]
# local_protocol = "https"

npx wrangler dev

# Then in a separate terminal, POST a simulated Zendesk payload:
curl -X POST http://localhost:8787 \
  -H "Content-Type: application/json" \
  -H "x-zendesk-webhook-signature: test" \
  -H "x-zendesk-webhook-signature-timestamp: $(date +%s)" \
  -d '{
    "ticket": {
      "id": "99999",
      "subject": "Test collaboration submission",
      "description": "Testing the TSANet outbound integration",
      "priority": "normal",
      "tags": ["tsanet_initiate"],
      "requester": {
        "name": "Test Agent",
        "email": "test@yourcompany.com",
        "phone": ""
      },
      "custom_fields": []
    }
  }'
```

{% hint style="info" %}
The signature check will fail during local development unless you compute a real HMAC. For initial testing, comment out the signature verification block in `verifyZendeskSignature` and restore it before deploying to production.
{% endhint %}

Once the worker returns `{"success": true, "token": "..."}`, tag a real test ticket with `tsanet_initiate` and verify:

1. The `tsanet_token` custom field is populated on the ticket
2. The `tsanet_initiate` tag is removed and `tsanet_open` is added
3. An internal comment appears confirming the collaboration was created
4. The case is visible in the TSANet Web App at connect.tsanet.org
{% endstep %}
{% endstepper %}

***

## Handling Multiple Partners Per Ticket

By default, the worker uses `TSANET_DEFAULT_DEPT_ID` — a single partner. For tickets that need to escalate to different partners, add a Zendesk custom field `TSANet Department ID` (text field). The worker reads this field first and falls back to the default only if it is empty.

To submit to multiple partners simultaneously, the agent tags the ticket multiple times — but Zendesk de-duplicates tags. A better pattern for multi-partner escalation is to use a dropdown custom field listing your configured partners, then submit separate collaboration cases for each. This is a Phase 2 improvement handled by the ZAF sidebar app.

***

## Environment Switching

When you are ready to go live, switch from Beta to Production:

```bash
# Update the environment variable
wrangler secret put TSANET_ENV
# Enter: PRODUCTION

# Update TSANet credentials to Production values
wrangler secret put TSANET_USERNAME
wrangler secret put TSANET_PASSWORD

# Redeploy
npx wrangler deploy
```

The worker will now hit `https://connect2.tsanet.org/v1` and `testSubmission` will be `false`, meaning real SLA clocks start and partners receive actual notifications.
