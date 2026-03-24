# Prerequisites

Confirm every item below before starting the integration. Missing prerequisites are the most common cause of lost time during setup.

---

## TSANet Requirements

### API Credentials

You need a dedicated TSANet API user — not a personal account. API users are excluded from system email notifications and are the correct credential type for integrations.

**To request credentials:**

1. Email **membership@tsanet.org** with the subject line: `API Credentials Request — Zendesk Integration`
2. Specify which environment you need: **Beta** (for development) or **Production** (for go-live)
3. TSANet will provision a user with Role: API and send you the username and password

{% hint style="warning" %}
Allow up to 2 business days for credential provisioning. Request Beta credentials first — you will need to validate your integration there before TSANet will provision Production credentials.
{% endhint %}

**Beta environment:** `https://connect2.tsanet.net`  
**Production environment:** `https://connect2.tsanet.org`

### Your Department ID

When submitting a collaboration to a partner, you need their `departmentId` (or `companyId` if they have no departments). Use the partner search endpoint to find it:

```bash
curl -H "Authorization: Bearer {your_token}" \
  "https://connect2.tsanet.net/v1/partners/{partner_name}"
```

You can also discover this from the TSANet Web App at [connect.tsanet.org](https://connect.tsanet.org) by viewing a partner's process form URL.

### Postman Collection

Download the TSANet Postman collection from the [Postman Collection](../api-reference/postman-collection.md) page. Use it to test all API calls manually before wiring them into the integration.

---

## Zendesk Requirements

### Account Type

{% hint style="danger" %}
**You must use a full Zendesk account — not a trial account.** Zendesk trial accounts are limited to 10 webhooks. ZIS Inbound Webhooks and your outbound webhook will use that limit immediately.
{% endhint %}

### Admin Access

You need Zendesk Admin access to:
- Create custom ticket fields
- Create and manage webhooks
- Create triggers and automations
- Create ZIS integrations and flows

### Custom Ticket Fields to Create

Create these fields in **Admin Center → Objects and rules → Tickets → Fields** before starting:

| Field Label | Field Type | Field Key | Visible to End Users |
|---|---|---|---|
| TSANet Token | Text | `tsanet_token` | No |
| TSANet Status | Dropdown | `tsanet_status` | No |
| TSANet Partner | Text | `tsanet_partner` | No |
| TSANet Respond By | Date/time | `tsanet_respond_by` | No |

{% hint style="info" %}
Note the numeric Field ID for `tsanet_token` after creating it — you will need it when configuring the ZIS flow and the Cloudflare Worker. The Field ID appears in the URL when you open the field in Admin Center (e.g., `admin_center/zendesk/fields/360012345678`).
{% endhint %}

### Zendesk Technology Partner Account

To register a ZIS integration, you need to be enrolled in the Zendesk developer program. Register at:

```
https://developer.zendesk.com
```

This is free and takes under 5 minutes.

---

## Cloudflare Requirements

The outbound worker runs on Cloudflare Workers. You need:

- A **free Cloudflare account** at [cloudflare.com](https://cloudflare.com) — the free tier allows 100,000 requests/day, which is far more than needed at typical TSANet collaboration volumes
- **Wrangler CLI** installed locally for deployment:

```bash
npm install -g wrangler
wrangler login
```

- **Node.js 18+** for local development and testing

---

## Summary Checklist

Before proceeding to setup:

- [ ] TSANet Beta credentials received (username + password for API user)
- [ ] TSANet `departmentId` identified for your target partner(s)
- [ ] TSANet Postman collection downloaded and tested: `POST /v1/login` returns a token
- [ ] Zendesk full account confirmed (not trial)
- [ ] Zendesk Admin access confirmed
- [ ] Four custom ticket fields created (`tsanet_token`, `tsanet_status`, `tsanet_partner`, `tsanet_respond_by`)
- [ ] `tsanet_token` Field ID noted
- [ ] Cloudflare account created
- [ ] Wrangler CLI installed and authenticated
