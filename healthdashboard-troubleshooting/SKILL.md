---
name: healthdashboard-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Health Dashboard
  problems by analyzing event interpretation, affected resources,
  scheduled changes, EventBridge integration, notification setup,
  organizational health, delegated administration, Health API usage,
  event filtering, incident response, and maintenance planning
  following structured runbooks. Activate when: event interpretation
  confusion, affected resource identification, scheduled maintenance
  planning, EventBridge integration issues, notification delivery
  failures, organizational health view problems, delegated admin
  setup, Health API errors, event filtering issues, incident response
  coordination, maintenance window planning, or the user says
  something is wrong with Health Dashboard without naming specific
  symptoms.
compatibility: >
  Requires AWS CLI or SDK access with Health API, EventBridge, IAM,
  Organizations, and optionally SNS and CloudWatch permissions.
  Business or Enterprise Support plan required for Health API access.
---

# Health Dashboard Diagnostics

## When to use

Any Health Dashboard investigation where the console alone is insufficient — event interpretation, affected resource identification, scheduled change management, EventBridge integration, organizational health, Health API access, or incident response coordination.

## Investigation workflow

### Step 1 — Collect and triage

```
aws health describe-events --filter '{"eventStatusCodes":["open","upcoming"]}' --region us-east-1
aws health describe-event-details --event-arns <event-arn> --region us-east-1
aws health describe-affected-entities --filter '{"eventArns":["<event-arn>"]}' --region us-east-1
aws health describe-event-types --filter '{"eventTypeCategories":["scheduledChange"]}' --region us-east-1
```

### Step 2 — Domain deep dive

```
aws health describe-events-for-organization --filter '{"eventStatusCodes":["open"]}' --region us-east-1
aws health describe-affected-accounts-for-organization --event-arn <event-arn> --region us-east-1
aws events list-rules --name-prefix Health --region us-east-1
aws health describe-event-aggregates --aggregate-field eventTypeCategory --filter '{"eventStatusCodes":["open"]}' --region us-east-1
```

Read `references/guardrails.md` before concluding on any Health Dashboard issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `health describe-events` | List health events by status, category, region |
| `health describe-event-details` | Full event description and timeline |
| `health describe-affected-entities` | Resources affected by an event |
| `health describe-event-types` | Available event type categories |
| `health describe-events-for-organization` | Org-wide health events |
| `health describe-affected-accounts-for-organization` | Accounts affected by org event |
| `health describe-event-aggregates` | Event counts by category |
| `events list-rules` | EventBridge rules for health events |

## Gotchas: Health Dashboard

- The Health API is only available in us-east-1 (and us-gov-west-1 for GovCloud). All API calls must target us-east-1.
- Health API requires Business or Enterprise Support plan. Basic and Developer plans can only use the console.
- Events have three categories: issue (service disruption), accountNotification (account-specific), and scheduledChange (planned maintenance).
- Organizational health events require Organizations with Health enabled as a trusted service.
- Health events are retained for 90 days. Events older than 90 days are not accessible via API.
- Scheduled changes include a start time and may include an end time. Not all scheduled changes have a fixed end time.
- EventBridge health events are published in us-east-1 for global events and in the affected region for regional events.
- Affected entities may not be immediately available when an event is first published. They are populated as AWS identifies impacted resources.
- Public events (service-wide) do not list specific affected entities. Only account-specific events list affected resources.
- Health Dashboard shows both public (service-wide) and account-specific events. Public events affect all customers in a region.

## Anti-hallucination rules

1. Always cite specific event ARNs, event details, affected entities, or event status as evidence.
2. Health API requires Business/Enterprise Support. Never suggest API calls for Basic/Developer plans.
3. The API endpoint is us-east-1 only. Never suggest using other regions for Health API calls.
4. Public events do not list specific affected entities. Never claim you can identify specific resources for public events.
5. Events are retained for 90 days only. Never suggest querying events older than 90 days.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 16 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Events | A1–A3 | Event interpretation, affected resources, scheduled changes |
| B — Notifications | B1–B2 | EventBridge integration, notification setup |
| C — Organization | C1–C2 | Organizational health, delegated admin |
| D — API | D1–D2 | Health API, event filtering |
| E — Response | E1–E2 | Incident response, maintenance planning |
| Z — Catch-All | Z1 | General troubleshooting |
