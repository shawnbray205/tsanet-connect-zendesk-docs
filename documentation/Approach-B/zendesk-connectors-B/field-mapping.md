# field mapping

This page is the definitive mapping between Zendesk ticket fields and the TSANet Connect 2.0/3.0 API model. Use it as the authoritative reference when building or debugging the integration.

***

## Outbound: Zendesk → TSANet

Fields sent when submitting a new collaboration request via `POST /v1/collaboration-requests`.

| Zendesk Source                    | TSANet Field                    | Notes                                                                                       |
| --------------------------------- | ------------------------------- | ------------------------------------------------------------------------------------------- |
| `ticket.id`                       | `internalCaseNumber`            | Stored as `submitterCaseNumber` in TSANet                                                   |
| `ticket.subject`                  | `problemSummary`                | Max 500 characters; truncate if needed                                                      |
| `ticket.description`              | `problemDescription`            | Max 5,000 characters                                                                        |
| `ticket.priority`                 | `priority`                      | See priority mapping table below                                                            |
| `ticket.requester.name`           | `submitterContactDetails.name`  | The agent's customer — or the agent themselves for B2B flows                                |
| `ticket.requester.email`          | `submitterContactDetails.email` | Must be a valid email; domain must match your TSANet account's authorized domains           |
| `ticket.requester.phone`          | `submitterContactDetails.phone` | Optional                                                                                    |
| `form.documentId`                 | `documentId`                    | Retrieved fresh from `GET /v1/forms/department/{id}` before every submission — do not cache |
| Custom field per form requirement | `customFields[].value`          | Map required form fields to Zendesk custom fields; see Custom Fields section below          |
| `false` in Production             | `testSubmission`                | `true` in Beta only; real SLA and notifications in Production                               |

### Priority Mapping

| Zendesk Priority | TSANet Priority | SLA      |
| ---------------- | --------------- | -------- |
| `urgent`         | `HIGH`          | 2 hours  |
| `high`           | `HIGH`          | 2 hours  |
| `normal`         | `MEDIUM`        | 4 hours  |
| `low`            | `LOW`           | 24 hours |

***

## Inbound: TSANet → Zendesk

Fields written to the Zendesk ticket when TSANet events are received via ZIS.

| TSANet Event Field                  | Zendesk Destination                        | How                                                                                    |
| ----------------------------------- | ------------------------------------------ | -------------------------------------------------------------------------------------- |
| `token`                             | Custom field: `tsanet_token`               | Written by outbound worker at case creation; used as lookup key for all inbound events |
| `status`                            | Custom field: `tsanet_status`              | Updated by ZIS flow on every status change event                                       |
| `status`                            | Ticket tag                                 | Tag format: `tsanet_{status_lowercase}` — e.g., `tsanet_accepted`                      |
| `respondBy`                         | Custom field: `tsanet_respond_by`          | ISO datetime of the SLA deadline                                                       |
| `partnerCompanyName`                | Custom field: `tsanet_partner`             | Display name of the collaborating partner                                              |
| `engineerName` + `engineerEmail`    | Internal comment                           | Included in the status change comment body                                             |
| `nextSteps`                         | Internal comment                           | Included in the acceptance comment body                                                |
| `note.summary` + `note.description` | Internal comment                           | Added by ZIS flow on `collaboration.note.created` events                               |
| `note.visibility`                   | Comment label prefix                       | `[TSANet — Partner Only]` or `[TSANet — Customer + Partner]`                           |
| _(SLA breach event)_                | Internal comment + tag `tsanet_sla_breach` | Added by ZIS flow on `collaboration.sla.breach` events                                 |

***

## Note Visibility Mapping

TSANet has three note visibility levels. Map them to Zendesk comment types:

| TSANet Visibility      | Zendesk Comment Type                                     | Comment Prefix                  |
| ---------------------- | -------------------------------------------------------- | ------------------------------- |
| `PARTNER_ONLY`         | Internal (private) comment                               | `[TSANet — Partner Only]`       |
| `CUSTOMER_AND_PARTNER` | Internal comment (recommend review before making public) | `[TSANet — Customer + Partner]` |
| `INTERNAL`             | Internal comment (not shared with partner)               | `[TSANet — Internal]`           |

{% hint style="warning" %}
`CUSTOMER_AND_PARTNER` notes are visible to the end customer in the TSANet context, but that does not automatically mean they should be public in Zendesk. An agent may need to decide whether to forward note content as a public reply to the customer. Always post as internal first and let the agent promote to public.
{% endhint %}

***

## Custom Fields

TSANet partners can configure custom required fields on their process forms — serial numbers, product versions, problem area dropdowns, etc. These must be included in your submission or the API returns a 400 error.

**Strategy for required custom fields:**

1. Retrieve the form via `GET /v1/forms/department/{departmentId}` and inspect `customFields` where `required: true`
2. For each required field, create a matching Zendesk custom ticket field
3. Train agents to fill these fields before tagging `tsanet_initiate`
4. The outbound worker reads these Zendesk fields and maps them to TSANet `customFields`

**Field type mapping:**

| TSANet Field Type               | Zendesk Field Type                                         |
| ------------------------------- | ---------------------------------------------------------- |
| `STRING`                        | Text                                                       |
| `SELECT`                        | Dropdown                                                   |
| Dynamic dropdown (parent/child) | Two linked dropdowns (requires Zendesk conditional fields) |

{% hint style="info" %}
The `adminNote` field returned by the form contains the partner's instructions for submitters — for example, "Please include the serial number and firmware version." Surface this note to the agent before they fill in the form. The outbound worker can include `adminNote` in the initial comment when the case is created.
{% endhint %}

***

## Zendesk Custom Fields Summary

| Label             | Key                 | Type      | Who Writes It    | Purpose                                        |
| ----------------- | ------------------- | --------- | ---------------- | ---------------------------------------------- |
| TSANet Token      | `tsanet_token`      | Text      | Outbound Worker  | Primary key linking ticket to TSANet case      |
| TSANet Status     | `tsanet_status`     | Dropdown  | ZIS Flow         | Current TSANet case status                     |
| TSANet Partner    | `tsanet_partner`    | Text      | ZIS Flow         | Collaborating partner company name             |
| TSANet Respond By | `tsanet_respond_by` | Date/time | ZIS Flow         | SLA response deadline                          |
| TSANet Dept ID    | `tsanet_dept_id`    | Text      | Agent (optional) | Override default partner department per ticket |

***

## Case Status Values

The `tsanet_status` dropdown field should have these options defined:

| Display Label            | Value         |
| ------------------------ | ------------- |
| Open — Awaiting Response | `OPEN`        |
| Information Requested    | `INFORMATION` |
| Accepted                 | `ACCEPTED`    |
| Rejected                 | `REJECTED`    |
| Closed                   | `CLOSED`      |

***

## TSANet Response Fields (on Acceptance)

When a partner accepts a collaboration, the TSANet approval response contains:

| Field           | Description                              | Where Surfaced                                                   |
| --------------- | ---------------------------------------- | ---------------------------------------------------------------- |
| `caseNumber`    | Partner's internal case number           | Included in acceptance comment                                   |
| `engineerName`  | Assigned partner engineer name           | Included in acceptance comment                                   |
| `engineerPhone` | Engineer's phone number                  | Included in acceptance comment                                   |
| `engineerEmail` | Engineer's email address                 | Included in acceptance comment                                   |
| `nextSteps`     | Partner's instructions back to your team | Included in acceptance comment — always surface this prominently |

The `receiverCaseNumber` is also stored on the TSANet case and available via `GET /v1/collaboration-requests/{token}`.
