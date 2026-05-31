---
title: "F1 — Data Retention Configuration"
description: "Diagnose data retention configuration and historical data access issues"
status: active
severity: MEDIUM
triggers: ["data missing", "historical data not available", "retention period", "old data gone"]
owner: devops-agent
objective: "Resolve data retention configuration and historical data access issues"
context: "RUM data has a retention period. Data beyond the retention period is not accessible. Long-term storage requires export to S3 or CloudWatch Logs."
---

## Phase 1 — Triage

MUST:
- Get app monitor configuration: `aws rum get-app-monitor --name <name>`
- Check data availability for the requested time range: `aws rum get-app-monitor-data --name <name> --time-range '{"After":<epoch-start>,"Before":<epoch-end>}' --max-results 5`
- Determine if the missing data falls outside the retention period

SHOULD:
- Check if data export to S3 or CloudWatch Logs is configured for long-term storage
- Verify the app monitor was active during the requested time period
- Check if the session sample rate was sufficient during the requested period

MAY:
- Check S3 or CloudWatch Logs for exported historical data
- Review when the app monitor was created to determine data availability start date

## Phase 2 — Remediate

MUST:
- If data is within retention: investigate collection issues (see B1 runbook)
- If data is beyond retention: check export destinations for archived data
- Document the data retention limitations for stakeholders

SHOULD:
- Configure data export to S3 for long-term retention if not already set up
- Set up CloudWatch metric export for key metrics that need long-term retention
- Plan data retention strategy based on compliance and analysis requirements

MAY:
- Configure S3 lifecycle rules for exported data management
- Set up Athena tables for querying exported historical data

## Common Issues

- symptoms: "RUM data available for recent days but not for last month"
  diagnosis: "Data may have exceeded the retention period."
  resolution: "RUM data retention is limited. For long-term data, configure export to S3 or CloudWatch Logs. Check if export was configured during the missing period."

## Safety Ratings

```
safety_ratings:
  - "get-app-monitor: GREEN — read-only monitor details"
  - "get-app-monitor-data: GREEN — read-only data retrieval"
  - "S3 data export configuration: YELLOW — enables data export"
  - "S3 lifecycle rules: YELLOW — affects data retention in S3"
```

## Escalation Conditions

- Critical historical data needed but beyond retention period and no export configured
- Data retention requirements change due to compliance regulations
- Export destination (S3/CloudWatch Logs) has its own retention issues
- Data volume from export exceeding storage budget

## Data Sensitivity

- `aws rum get-app-monitor-data`: MEDIUM — contains user session data
- Exported RUM data in S3: MEDIUM — contains historical user session and performance data
- Data retention configuration: LOW — operational configuration

## Prohibited Actions

- NEVER promise historical data recovery if it is beyond the retention period and not exported
- NEVER modify S3 lifecycle rules without understanding impact on other data in the bucket
- NEVER delete exported data without verifying compliance requirements
- NEVER assume all historical data is available without checking retention limits

## Phase 3 — Rollback

1. If data export was configured in error, disable it
2. If S3 lifecycle rules were changed, restore original rules
3. Verify current data collection is unaffected by retention configuration changes
4. Document the data retention investigation and findings
5. Confirm stakeholders understand data availability limitations

## Output Format

```yaml
root_cause: "data_retention — <specific_cause>"
evidence:
  - type: retention_issue
    content: "<retention configuration and data availability details>"
severity: MEDIUM
mitigation:
  immediate: "Check export destinations for historical data"
  long_term: "Implement comprehensive data export and retention strategy"
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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
