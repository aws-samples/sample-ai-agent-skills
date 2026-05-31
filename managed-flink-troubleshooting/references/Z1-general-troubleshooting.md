---
title: "Z1 — General Managed Flink Troubleshooting"
description: "Catch-all runbook for Managed Flink issues"
status: active
severity: MEDIUM
triggers: ["Managed Flink error", "Kinesis Analytics error", "Flink application not working"]
owner: devops-agent
objective: "Systematic investigation of unclassified Managed Flink issues"
context: "Use when the issue doesn't clearly match a specific runbook."
---

## Phase 1 — Triage

MUST:
- List applications: `aws kinesisanalyticsv2 list-applications`
- Describe the application: `aws kinesisanalyticsv2 describe-application --application-name <app-name>`
- Review CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=kinesisanalytics.amazonaws.com --max-results 20`

SHOULD:
- Check application status and uptime metrics
- Review CloudWatch Logs for errors: `aws logs filter-log-events --log-group-name /aws/kinesis-analytics/<app-name> --start-time <epoch-ms> --limit 50`

MAY:
- Check AWS Health Dashboard for Managed Flink service issues
- Review CloudWatch metrics dashboard for anomalies

## Phase 2 — Remediate

MUST:
- Identify problem domain and pivot to specific runbook

SHOULD:
- Document findings and investigation path

MAY:
- Set up CloudWatch alarms for application health metrics

## Safety Ratings

```
safety_ratings:
  - "list-applications: GREEN — read-only application enumeration"
  - "describe-application: GREEN — read-only application details"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
  - "logs filter-log-events: GREEN — read-only log review"
```

## Escalation Conditions

- Managed Flink service appears non-functional in the region
- Application fails with internal service errors not related to user code
- CloudTrail shows unauthorized API calls to Managed Flink
- Issue cannot be classified into any specific runbook after systematic investigation

## Data Sensitivity

- `aws kinesisanalyticsv2 list-applications`: LOW — returns application metadata
- `aws kinesisanalyticsv2 describe-application`: MEDIUM — reveals application configuration
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history
- `aws logs filter-log-events`: HIGH — may contain application data

## Prohibited Actions

- NEVER delete applications during general troubleshooting
- NEVER modify application configuration without first identifying the root cause
- NEVER dismiss the issue without documenting the investigation path
- NEVER assume the issue is transient without checking metrics and logs

## Phase 3 — Rollback

1. If any configuration changes were made during investigation, revert them
2. Restore application settings if they were modified during troubleshooting
3. Verify application is processing data correctly
4. Document all investigation steps and findings
5. Verify the Managed Flink service is functioning normally before closing

## Common Issues

- symptoms: "Application status shows RUNNING but no output is produced"
  diagnosis: "Application is running but may have stale watermarks, empty source, or sink errors."
  resolution: "Check watermark metrics, source record count, and sink error logs. Pivot to specific runbook."

## Output Format

```yaml
root_cause: "general_managed_flink — <specific_cause>"
evidence:
  - type: investigation_summary
    content: "<findings>"
severity: MEDIUM
mitigation:
  immediate: "Address identified issue"
  long_term: "Implement monitoring"
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
