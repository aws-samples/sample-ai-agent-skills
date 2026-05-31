---
title: "B3 — Batch Write Errors"
description: "Diagnose Timestream batch write operation failures"
status: active
severity: MEDIUM
triggers:
  - "batch write failed"
  - "WriteRecords batch error"
  - "partial write failure"
  - "batch size exceeded"
owner: devops-agent
objective: "Identify and resolve batch write operation issues"
context: "Timestream WriteRecords accepts batches of up to 100 records per call. Batch writes can partially fail (some records accepted, others rejected). Common issues include batch size limits, throttling, and mixed record types."
---

## Phase 1 — Triage

MUST:
- Verify batch size is within limits (max 100 records per WriteRecords call)
- Check for partial failures in the WriteRecords response
- Check CloudWatch for write throttling: `aws cloudwatch get-metric-statistics --namespace AWS/Timestream --metric-name SuccessfulRequestLatency --dimensions Name=DatabaseName,Value=<database> Name=Operation,Value=WriteRecords --start-time <start> --end-time <end> --period 300 --statistics Average`
- Verify all records in the batch have consistent dimensions and measure types

SHOULD:
- Check if records mix single-measure and multi-measure formats
- Verify the total payload size is within limits
- Check for throttling errors in application logs

MAY:
- Analyze write patterns for optimization opportunities
- Check account-level write throughput quotas

## Phase 2 — Remediate

MUST:
- Limit batch size to 100 records per WriteRecords call
- Handle partial failures by retrying only rejected records
- Ensure consistent record format within each batch

SHOULD:
- Use multi-measure records for efficiency (fewer API calls)
- Implement exponential backoff for throttling
- Group records by common dimensions for optimal batching

MAY:
- Request write throughput quota increase
- Implement write buffering and batching in the application

## Common Issues

- symptoms: "ValidationException — too many records"
  diagnosis: "Batch exceeds 100 records per WriteRecords call."
  resolution: "Split batches into chunks of 100 records or fewer."

- symptoms: "Partial batch failure with some records rejected"
  diagnosis: "Some records have invalid format or timestamps."
  resolution: "Check RejectedRecords in the response. Fix and retry only failed records."

- symptoms: "ThrottlingException on batch writes"
  diagnosis: "Write throughput exceeds account or table limits."
  resolution: "Implement backoff/retry. Reduce write frequency or request quota increase."

## Output Format

```yaml
root_cause: "batch_write_error — <specific_cause>"
evidence:
  - type: batch_size
    content: "<records per batch>"
  - type: rejected_records
    content: "<rejected records from response>"
  - type: throttling
    content: "<throttling metrics>"
severity: MEDIUM
mitigation:
  immediate: "Fix batch size and handle partial failures"
  long_term: "Optimize write pipeline with proper batching and retries"
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
