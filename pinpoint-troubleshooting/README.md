# Amazon Pinpoint Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon Pinpoint problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon Pinpoint when the console alone isn't enough — campaign delivery, segment management, email/SMS/push channels, journey orchestration, template management, deliverability optimization, and event analytics.

### Activate When

- Campaign delivery failures or low delivery rates
- Segment creation or dynamic segment issues
- Email bounces, complaints, or deliverability problems
- SMS delivery failures or phone number issues
- Push notification configuration or delivery errors
- Journey execution failures or unexpected behavior
- Template rendering issues
- Event stream or analytics problems
- Phone number provisioning failures
- Opt-out or consent management issues
- General Pinpoint errors without clear symptoms

---

## Skill Structure

```
pinpoint-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-campaign-creation.md
    ├── A2-campaign-delivery.md
    ├── A3-campaign-scheduling.md
    ├── B1-segment-creation.md
    ├── B2-dynamic-segments.md
    ├── C1-email-channel.md
    ├── C2-email-bounces.md
    ├── C3-email-deliverability.md
    ├── D1-sms-delivery.md
    ├── D2-phone-number-issues.md
    ├── E1-push-config.md
    ├── E2-push-delivery.md
    ├── F1-journey-execution.md
    ├── F2-journey-activities.md
    ├── G1-template-rendering.md
    ├── H1-event-streams.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (16 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Campaigns** | A1–A3 | Campaign creation, delivery failures, scheduling |
| **B — Segments** | B1–B2 | Segment creation, dynamic segment issues |
| **C — Email** | C1–C3 | Email channel, bounces, deliverability |
| **D — SMS** | D1–D2 | SMS delivery, phone number issues |
| **E — Push** | E1–E2 | Push notification config, delivery failures |
| **F — Journeys** | F1–F2 | Journey execution, activity failures |
| **G — Templates** | G1 | Template rendering issues |
| **H — Analytics** | H1 | Event stream and analytics |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/guardrails.md` covering email identity verification, SMS phone number provisioning, 10DLC registration, campaign scheduling timezone, segment eventual consistency, journey duration limits, push platform credentials, event stream IAM, SMS spending limits, endpoint quotas, channel-specific opt-out, and sandbox restrictions.

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `mobiletargeting:*`, `sms-voice:*`, `ses:*`, `cloudwatch:GetMetricStatistics`, `cloudtrail:LookupEvents`
- Verified email identities for email channel
- Provisioned phone numbers for SMS channel

---

## License

MIT-0
