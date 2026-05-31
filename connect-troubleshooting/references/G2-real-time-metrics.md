---
title: "G2 — Real-Time Metrics Issues"
description: "Diagnose issues with real-time metrics and dashboards"
status: active
severity: MEDIUM
triggers:
  - "Real-time metrics wrong"
  - "Dashboard not updating"
  - "Agent count incorrect"
  - "Queue metrics stale"
owner: devops-agent
objective: "Fix real-time metrics accuracy and availability"
context: "Real-time metrics show current state: agents available, contacts in queue, oldest contact age, etc. They update every 15 seconds. Issues include: stale data, incorrect counts, missing queues, or API errors."
---

## Phase 1 — Triage

MUST:
- Check real-time metrics via API: `aws connect get-current-metric-data`
- Compare API results with console dashboard
- Verify the queue and routing profile filters are correct
- Check if the metrics match the actual agent and queue state

SHOULD:
- Verify the instance ID and queue ARNs are correct
- Check if the dashboard has the correct filters applied
- Verify agent states match what's shown in metrics

MAY:
- Check CloudWatch for Connect service metrics
- Compare with historical metrics for consistency

## Phase 2 — Remediate

MUST:
- Fix filter configuration if metrics show wrong queues/agents
- Refresh the dashboard if data is stale
- Verify the API call parameters are correct

SHOULD:
- Set up automated metric collection for monitoring
- Implement metric alerting for key thresholds

MAY:
- Build custom dashboards using the metrics API
- Implement metric data validation

## Common Issues

- symptoms: "Dashboard shows 0 agents but agents are logged in"
  diagnosis: "Dashboard filter doesn't include the correct queues or routing profiles."
  resolution: "Update dashboard filters to include all relevant queues."

- symptoms: "Metrics don't update"
  diagnosis: "Dashboard auto-refresh disabled or API polling stopped."
  resolution: "Enable auto-refresh. Check API polling interval."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Delete/Remove resources: RED - Potentially irreversible, requires confirmation"
```

## Escalation Conditions

- Contact center is live with active calls or chats in progress
- Issue affects agent ability to receive or handle contacts
- Fix requires modifying a contact flow associated with a phone number
- Resolution involves changes to hours of operation during business hours
- Issue impacts call recording, CTR generation, or compliance requirements

## Data Sensitivity

HIGH - Contact Trace Records (CTRs) contain customer phone numbers, call timestamps, and interaction details. Call recordings contain customer voice data. Chat transcripts may contain PII, financial data, or healthcare information. Agent data includes employee performance metrics.

## Prohibited Actions

- NEVER suggest deleting a contact flow that is associated with a phone number - this breaks inbound routing
- NEVER suggest modifying production hours of operation during active business hours
- NEVER recommend removing security profiles from agents while they are logged in
- NEVER suggest disabling call recording on queues that have compliance requirements
- NEVER recommend deleting or modifying a published contact flow without first testing in a non-production flow

## Phase 3 - Rollback

1. If queue configuration was changed, restore original routing profile assignments and queue settings
2. If agent routing or security profiles were modified, restore original profile assignments
## Output Format

```yaml
root_cause: "real_time_metrics — <specific issue>"
evidence:
  - type: metrics_api
    content: "<metric values and expected values>"
severity: MEDIUM
mitigation:
  immediate: "Fix metric filters or refresh"
  long_term: "Implement automated metric monitoring"
```

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling call recording without compliance review"
  - "NEVER suggest removing authentication"
  - "NEVER expose customer PII in logs"
