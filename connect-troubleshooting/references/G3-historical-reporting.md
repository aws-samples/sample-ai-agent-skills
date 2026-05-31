---
title: "G3 — Historical Reporting Issues"
description: "Diagnose issues with historical metrics and reporting"
status: active
severity: MEDIUM
triggers:
  - "Historical metrics wrong"
  - "Report data missing"
  - "Metrics don't match"
  - "Reporting delay"
owner: devops-agent
objective: "Fix historical reporting accuracy and completeness"
context: "Historical metrics aggregate contact data over time periods. They have a 24-hour processing delay for some aggregations. Data retention is 24 months for most metrics. Issues include: data gaps, incorrect aggregations, timezone mismatches, or filter errors."
---

## Phase 1 — Triage

MUST:
- Check the time range and timezone of the report
- Verify the metric definitions match what's expected
- Check if the data gap is within the 24-hour processing window
- Verify queue and agent filters are correct

SHOULD:
- Compare with real-time metrics for recent periods
- Check if the issue affects specific metrics or all metrics
- Verify the report grouping (by queue, agent, routing profile)

MAY:
- Use the GetMetricDataV2 API for programmatic verification
- Check if CTR data matches the reported metrics

## Phase 2 — Remediate

MUST:
- Fix report filters and timezone settings
- Wait for the processing delay if data is recent
- Verify metric definitions match the intended measurement

SHOULD:
- Document report configurations and expected values
- Cross-reference with CTR data for validation

MAY:
- Build custom reports using the metrics API
- Implement report validation automation

## Common Issues

- symptoms: "Yesterday's data not showing in reports"
  diagnosis: "Normal processing delay — some metrics take up to 24 hours."
  resolution: "Wait for processing. Use real-time metrics for recent data."

- symptoms: "Metrics don't match between reports"
  diagnosis: "Different timezone settings or filter configurations."
  resolution: "Standardize timezone and filter settings across reports."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
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
3. If phone number associations were changed, reassociate the number with the original contact flow
## Output Format

```yaml
root_cause: "historical_reporting — <specific issue>"
evidence:
  - type: report_config
    content: "<report settings and data>"
severity: MEDIUM
mitigation:
  immediate: "Fix report configuration"
  long_term: "Standardize reporting and implement validation"
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
