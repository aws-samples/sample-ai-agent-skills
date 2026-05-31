---
title: "E2 — Watermark Issues"
description: "Diagnose Managed Flink watermark and event-time processing issues"
status: active
severity: MEDIUM
triggers: ["watermark not advancing", "late data", "event time error", "window not firing"]
owner: devops-agent
objective: "Resolve watermark issues including stale watermarks, late data handling, and window trigger problems"
context: "Watermarks track event-time progress. Windows fire based on watermarks. Stale watermarks prevent windows from closing and producing results."
---

## Phase 1 — Triage

MUST:
- Check watermark metric: `aws cloudwatch get-metric-statistics --namespace AWS/KinesisAnalytics --metric-name currentOutputWatermark --dimensions Name=Application,Value=<app-name> --start-time <start> --end-time <end> --period 60 --statistics Minimum`
- Check if watermark is advancing over time
- Review application logs for watermark-related warnings: `aws logs filter-log-events --log-group-name /aws/kinesis-analytics/<app-name> --filter-pattern "watermark" --start-time <epoch-ms> --limit 50`

SHOULD:
- Check input watermark vs output watermark gap
- Verify event timestamps in source data are reasonable

MAY:
- Check for idle source partitions that prevent watermark advancement
- Review window operator output metrics

## Phase 2 — Remediate

MUST:
- Fix watermark strategy to match data characteristics
- Handle idle source partitions with withIdleness() configuration
- Ensure event timestamps are extracted correctly from source records

SHOULD:
- Configure allowed lateness for windows to handle late data
- Implement side output for late data collection

MAY:
- Use processing time instead of event time if event timestamps are unreliable
- Implement custom watermark generators for irregular data patterns

## Safety Ratings

```
safety_ratings:
  - "cloudwatch get-metric-statistics: GREEN — read-only metrics"
  - "logs filter-log-events: GREEN — read-only log review"
  - "update-application (watermark config): YELLOW — modifies time semantics"
  - "create-application-snapshot: YELLOW — captures state before changes"
```

## Escalation Conditions

- Watermarks are permanently stuck and windows never fire
- Late data volume exceeds acceptable thresholds
- Watermark strategy change requires application restart and state migration
- Event timestamps in source data are inconsistent or unreliable

## Data Sensitivity

- `aws cloudwatch get-metric-statistics`: LOW — aggregated metrics only
- `aws logs filter-log-events`: MEDIUM — may contain event timestamps and data patterns
- Watermark configuration: LOW — operational settings

## Prohibited Actions

- NEVER ignore stale watermarks — they prevent windows from producing results
- NEVER switch from event time to processing time without understanding semantic changes
- NEVER set allowed lateness to infinity — it causes unbounded state growth
- NEVER assume all source partitions produce data at the same rate

## Phase 3 — Rollback

1. Create snapshot before watermark changes
2. If watermark strategy change caused issues, revert code and restore from snapshot
3. Verify windows fire correctly after rollback
4. Check for data gaps in window output during the issue period
5. Document watermark strategy and late data handling

## Common Issues

- symptoms: "Windows never produce output despite data flowing through the application"
  diagnosis: "Watermark is not advancing. An idle source partition holds back the global watermark."
  resolution: "Configure withIdleness() on the source to mark idle partitions. Set idle timeout appropriate for data arrival pattern."

## Output Format

```yaml
root_cause: "watermark — <specific_cause>"
evidence:
  - type: watermark_metric
    content: "<metric details>"
severity: MEDIUM
mitigation:
  immediate: "Fix watermark issue"
  long_term: "Monitor watermark health"
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
