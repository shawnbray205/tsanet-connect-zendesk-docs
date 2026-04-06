# SUMMARY

## Proposed page hierarchy for tsanet.gitbook.io/connect

Place these pages under the existing connector section in the GitBook sidebar, parallel to the Salesforce and Dynamics connector entries.

## Sidebar Entry

```
zendesk  Zendesk Connector

  + Overview                   → /documentation/zendesk-connector/README.md
  + Prerequisites              → /documentation/zendesk-connector/prerequisites.md
  + Inbound Setup (ZIS)        → /documentation/zendesk-connector/inbound-setup.md
  + Outbound Setup (Worker)    → /documentation/zendesk-connector/outbound-setup.md
  + Field Mapping Reference    → /documentation/zendesk-connector/field-mapping.md
  + Testing Checklist          → /documentation/zendesk-connector/testing-checklist.md
  + Email-to-Case Bridge       → /documentation/zendesk-connector/email-to-case.md
```

***

## Files in This Package

| File                   | GitBook Page            | Word Count (approx) |
| ---------------------- | ----------------------- | ------------------: |
| `README.md`            | Overview                |               \~500 |
| `prerequisites.md`     | Prerequisites           |               \~600 |
| `inbound-setup.md`     | Inbound Setup (ZIS)     |             \~1,800 |
| `outbound-setup.md`    | Outbound Setup (Worker) |             \~2,200 |
| `field-mapping.md`     | Field Mapping Reference |               \~800 |
| `testing-checklist.md` | Testing Checklist       |             \~1,400 |
| `email-to-case.md`     | Email-to-Case Bridge    |               \~700 |

***

## Integration Guide Cross-Links to Add

In the existing Integration Guide page (`/api-reference/integration-guide`), add Zendesk to the TSANet Supported Connectors section:

```markdown
**Zendesk:** See [Zendesk Connector in Documentation](/documentation/zendesk-connector/)
```

In the Integration Guide's connector decision matrix, add a row:

| System  | Path                    | Docs                                                          |
| ------- | ----------------------- | ------------------------------------------------------------- |
| Zendesk | ZIS + Cloudflare Worker | [Zendesk Connector](file:///documentation/zendesk-connector/) |

***

## Changelog Entry to Add

In `/changelog`, under a new release entry:

```markdown
## May 2026

### Zendesk Connector — Community Integration Guide

- Published integration reference guide for Zendesk Support
- Architecture: Zendesk Integration Services (ZIS) for inbound events + Cloudflare Worker for outbound case submission
- Includes: ZIS flow template, Cloudflare Worker source code, field mapping reference, step-by-step testing checklist, and Email-to-Case bridge for zero-code option
- Sample code available at: https://github.com/tsanetgit/zendesk-sample
```
