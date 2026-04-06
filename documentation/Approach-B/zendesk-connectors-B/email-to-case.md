# email to case

If you are not ready to deploy the full ZIS + Worker integration, this page gets your team functional in under an hour using Zendesk's native Email-to-Case. You will work TSANet cases manually from the TSANet Web App, but all notifications arrive in Zendesk as tickets, keeping your team in one place.

This approach is a valid long-term choice for low-volume members. The typical TSANet inbound volume is approximately 3 requests per month — for many teams, a lightweight email flow is sufficient indefinitely.

## How It Works

TSANet sends all collaboration notifications from a single address: **connect@tsanet.org**

When a partner submits a collaboration request to you, TSANet emails your registered address. When case status changes, notes are added, or SLA breaches occur, TSANet emails again. If you route these emails into Zendesk, your agents can track and respond to collaborations from their normal Zendesk queue — and use the TSANet Web App only for the actual accept/reject/note actions.

```
connect@tsanet.org  →  your TSANet email alias  →  Zendesk Email-to-Case
                                                          │
                                                          ▼
                                                    Zendesk Ticket
                                                    (tagged: tsanet_inbound)
                                                          │
                                                          ▼
                                                    Agent works case in
                                                    TSANet Web App
                                                    connect.tsanet.org
```

{% stepper %}
{% step %}
### Configure Your TSANet Inbound Process Email

In the TSANet Web App under **Admin → Inbound Process**, set your **To Email** to a dedicated address you control — for example:

```
tsanet-cases@yourcompany.com
```

This address will receive all TSANet collaboration notifications. Do not use a personal inbox.
{% endstep %}

{% step %}
### Configure Zendesk Email-to-Case

In **Admin Center → Channels → Email**, add a new support address:

| Setting          | Value                                                              |
| ---------------- | ------------------------------------------------------------------ |
| Display name     | TSANet Collaboration Cases                                         |
| Email address    | tsanet-cases@yourcompany.zendesk.com (or forward from your domain) |
| Default assignee | Your TSANet-handling group or agent                                |

If you use your own domain (`tsanet-cases@yourcompany.com`), set up email forwarding from that address to your Zendesk support address.
{% endstep %}

{% step %}
### Allowlist TSANet's Sending Address

Zendesk may initially treat emails from connect@tsanet.org as spam or reject them as external. Ensure the address is accepted:

**In Admin Center → Security → Allowlist and blocklist:**

Add `connect@tsanet.org` to the allowlist.

Also confirm with your IT team that `connect@tsanet.org` is not blocked at the mail gateway level. TSANet uses Mandrill for email delivery — some corporate mail filters block Mandrill-originated mail by default.
{% endstep %}

{% step %}
### Create a Zendesk Trigger to Tag Inbound TSANet Tickets

Create a trigger that identifies emails from TSANet and tags them appropriately.

**In Admin Center → Objects and rules → Business rules → Triggers → Add trigger:**

**Trigger name:** `TSANet — Tag Inbound Cases`

**Conditions (ALL):**

* Received at — `tsanet-cases@yourcompany.zendesk.com`
* Ticket is — Created

**Actions:**

* Add tags — `tsanet_inbound`
* Assign to — _(your TSANet handling group)_
* Set priority — _(High — these are collaboration requests from partners)_
{% endstep %}

{% step %}
### Create a Zendesk View for TSANet Cases

In **Admin Center → Workspaces → Views → Add view:**

| Setting    | Value                                 |
| ---------- | ------------------------------------- |
| View name  | TSANet Collaboration Cases            |
| Conditions | Tags — Contains — `tsanet_inbound`    |
| Columns    | Ticket ID, Subject, Created, Assignee |
| Group by   | None                                  |
| Order by   | Created (descending)                  |

Your agents now have a dedicated queue for all TSANet collaboration notifications.
{% endstep %}
{% endstepper %}

## Agent Workflow

Once a TSANet email arrives as a Zendesk ticket:

1. Agent opens the ticket and reads the collaboration request details in the email body
2. Agent navigates to [connect.tsanet.org](https://connect.tsanet.org/) and logs in
3. Agent locates the case (it will be in the **Inbound** queue)
4. Agent accepts, rejects, or requests more information from the Web App
5. Agent adds any internal notes in Zendesk for context
6. When the collaboration closes, agent marks the Zendesk ticket as solved

The Zendesk ticket serves as the record in your system; the TSANet Web App is where the actual collaboration actions happen.

## Limitations of This Approach

| Limitation                               | Impact                                                                                                 |
| ---------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| No automatic token linkage               | You cannot automatically link the TSANet case to a customer's Zendesk ticket unless you do it manually |
| Status changes arrive as separate emails | Each update creates a new Zendesk ticket unless Zendesk's email threading groups them correctly        |
| No SLA visibility in Zendesk             | TSANet SLA deadlines are only visible in the Web App, not in Zendesk                                   |
| Manual accept/reject                     | Every response requires navigating to the Web App                                                      |
| No note bidirectionality                 | Notes added in Zendesk do not automatically sync to the TSANet collaboration                           |

If these limitations become friction points as your volume or process matures, the full [ZIS + Worker integration](inbound-setup.md) resolves all of them.

## Improving Email Threading

By default, Zendesk creates a new ticket for each TSANet email, even if they are about the same collaboration case. To improve threading:

1. In your TSANet inbound process settings, enable the **TSANet Token in Subject** email template. This adds the collaboration token to every email subject line (e.g., `[TSANet-abc123xyz] Collaboration Update`).
2. Create a Zendesk trigger that extracts the token from the subject and adds it as a tag (`tsanet_token_abc123xyz`). Agents can then search for all tickets related to the same collaboration.

This is not a full solution — Zendesk's threading works best when the `Message-ID` header matches, which TSANet emails will not always provide — but it significantly improves the agent experience.
