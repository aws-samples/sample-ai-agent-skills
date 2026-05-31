---
title: "Z1 — General Timestream Troubleshooting"
description: "Catch-all runbook for Timestream issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers:
  - "Timestream not working"
  - "Timestream problem"
  - "Timestream issue"
  - "something wrong with Timestream"
owner: devops-agent
objective: "Systematically diagnose unspecified Timestream issues"
context: "This catch-all runbook provides a systematic approach when the specific issue is unclear. It covers the most common Timestream problems and guides toward the appropriate specific runbook."
---

## Phase 1 — Triage

MUST:
- Check database status: `aws timestream-write list-databases`
- Check table status: `aws timestream-write list-tables --database-name <database>`
- Check table details: `aws timestream-write describe-table --database-name <database> --table-name <table>`
- Check CloudWatch for errors: `aws cloudwatch get-metric-statistics --namespace AWS/Timestream --metric-name UserErrors --dimensions Name=DatabaseName,Value=<database> --start-time <start> --end-time <end> --period 300 --statistics Sum`

SHOULD:
- Verify endpoint discovery: `aws timestream-write describe-endpoints`
- Check for system errors: `aws cloudwatch get-metric-statistics --namespace AWS/Timestream --metric-name SystemErrors --dimensions Name=DatabaseName,Value=<database> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Review CloudTrail for recent events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=timestream.amazonaws.com --max-results 20`

MAY:
- Check IAM permissions
- Verify VPC endpoint configuration
- Test a simple write and query

## Phase 2 — Remediate

MUST:
- Address the identified issue based on triage findings
- Redirect to the appropriate specific runbook

SHOULD:
- Redirect based on findings:
  - Table issues → A1-A2
  - Write issues → B1-B3
  - Query issues → C1-C3
  - Memory/magnetic issues → D1-D2
  - Integration issues → E1-E2
  - Security issues → F1-F2

MAY:
- Perform a full Timestream configuration audit
- Document findings

## Common Issues

- symptoms: "General Timestream not working"
  diagnosis: "Multiple possible causes — follow systematic triage."
  resolution: "Check database/table status, errors, endpoints, permissions in order."

- symptoms: "Timestream was working but stopped"
  diagnosis: "Recent configuration change, permission change, or endpoint issue."
  resolution: "Check CloudTrail for recent changes. Verify endpoints and permissions."

- symptoms: "Some operations work but others fail"
  diagnosis: "Partial IAM permissions or endpoint configuration issue."
  resolution: "Check permissions for specific operations (write vs query)."

## Output Format

```yaml
root_cause: "general_timestream_issue — <specific_cause>"
evidence:
  - type: database_status
    content: "<database and table status>"
  - type: error_metrics
    content: "<CloudWatch error metrics>"
  - type: endpoint_status
    content: "<endpoint discovery result>"
severity: MEDIUM
mitigation:
  immediate: "Address the identified issue"
  long_term: "Implement comprehensive Timestream monitoring"
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
