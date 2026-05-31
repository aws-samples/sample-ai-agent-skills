---
title: "B1 — Write Failures"
description: "Diagnose Timestream write ingestion failures"
status: active
severity: HIGH
triggers:
  - "write failed"
  - "ingestion error"
  - "WriteRecords error"
  - "cannot write to Timestream"
owner: devops-agent
objective: "Identify and resolve write ingestion failures"
context: "Write failures can be caused by IAM permissions, endpoint discovery issues, invalid record format, throttling, or table configuration. Timestream requires endpoint discovery before writing."
---

## Phase 1 — Triage

MUST:
- Check IAM permissions: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names timestream:WriteRecords timestream:DescribeEndpoints`
- Verify the table exists and is active: `aws timestream-write describe-table --database-name <database> --table-name <table>`
- Check CloudWatch for write errors: `aws cloudwatch get-metric-statistics --namespace AWS/Timestream --metric-name UserErrors --dimensions Name=DatabaseName,Value=<database> Name=TableName,Value=<table> Name=Operation,Value=WriteRecords --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Verify endpoint discovery works: `aws timestream-write describe-endpoints`

SHOULD:
- Check for throttling: look for ThrottlingException in application logs
- Verify record format (dimensions, measure name, measure value, time, time unit)
- Check if the write is within the memory store retention window

MAY:
- Check CloudTrail for WriteRecords events
- Verify network connectivity to Timestream endpoints

## Phase 2 — Remediate

MUST:
- Fix IAM permissions for timestream:WriteRecords and timestream:DescribeEndpoints
- Ensure endpoint discovery is working (SDK handles automatically)
- Fix record format to match Timestream requirements

SHOULD:
- Implement retry logic with exponential backoff for throttling
- Validate records before sending to Timestream
- Use batch writes for efficiency

MAY:
- Request write throughput quota increase
- Monitor write latency and error rates

## Common Issues

- symptoms: "AccessDeniedException on WriteRecords"
  diagnosis: "IAM role missing timestream:WriteRecords permission."
  resolution: "Add timestream:WriteRecords and timestream:DescribeEndpoints to the role."

- symptoms: "ValidationException — invalid record"
  diagnosis: "Record format incorrect (missing required fields or invalid types)."
  resolution: "Ensure each record has dimensions, measure name, measure value, time, and time unit."

- symptoms: "ThrottlingException on writes"
  diagnosis: "Write throughput exceeds table or account limits."
  resolution: "Implement backoff/retry. Request quota increase if needed."

## Output Format

```yaml
root_cause: "write_failure — <specific_cause>"
evidence:
  - type: iam_permissions
    content: "<IAM evaluation>"
  - type: table_status
    content: "<table state>"
  - type: error_metrics
    content: "<CloudWatch error metrics>"
severity: HIGH
mitigation:
  immediate: "Fix the specific write blocker"
  long_term: "Implement robust write pipeline with retries and monitoring"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Time-series data may contain IoT sensor readings and telemetry
- Query results may reveal operational patterns and metrics
- Scheduled query outputs may aggregate sensitive measurements
- Classification: CONFIDENTIAL - operational telemetry and measurement data

## Prohibited Actions

- NEVER delete Timestream tables containing production time-series data
- NEVER reduce memory store retention below the current data window
- NEVER modify magnetic store retention without data lifecycle review
- NEVER cancel scheduled queries that feed production dashboards

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

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
