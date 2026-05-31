---
title: "B2 — Custom Source Issues"
description: "Diagnose custom log source registration and delivery problems"
status: active
severity: HIGH
triggers:
  - "custom source failed"
  - "custom log delivery"
  - "OCSF format error"
  - "custom source registration"
owner: devops-agent
objective: "Resolve custom source registration and data delivery issues"
context: "Custom sources must be registered before delivering data. Data must be in OCSF-compliant Apache Parquet format delivered to the assigned S3 prefix. Non-compliant data is rejected."
---

## Phase 1 — Triage

MUST:
- List custom sources: `aws securitylake list-log-sources --sources customLogSources`
- Verify source is registered: `aws securitylake get-data-lake-sources --accounts <account-id>`
- Check S3 prefix for delivered data: `aws s3 ls s3://<bucket>/ext/<custom-source-name>/`
- Verify Parquet files conform to OCSF schema
- Check exceptions: `aws securitylake list-data-lake-exceptions --regions <region>`

SHOULD:
- Validate OCSF schema compliance of sample data
- Check Glue crawler has run for the custom source table
- Verify IAM role used for delivery has S3 write permissions

## Phase 2 — Remediate

MUST:
- Register custom source first: `aws securitylake create-custom-log-source --source-name <name> --configuration ...`
- Deliver data in OCSF-compliant Apache Parquet format
- Use the S3 prefix assigned during registration
- Ensure correct OCSF event class for the data type

SHOULD:
- Test with small sample data before full delivery
- Monitor Glue table for new partitions after delivery

## Output Format

```yaml
root_cause: "custom-source — <specific_cause>"
evidence:
  - type: source_registration
    content: "<registration status>"
  - type: data_format
    content: "<Parquet/OCSF compliance>"
severity: HIGH
mitigation:
  immediate: "Fix registration or data format"
  long_term: "Implement OCSF validation in delivery pipeline"
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

- Security logs contain authentication and authorization events
- Network flow logs may reveal infrastructure topology
- CloudTrail data contains API activity across all services
- Classification: HIGHLY CONFIDENTIAL - aggregated security telemetry requires strict access control

## Prohibited Actions

- NEVER delete or disable Security Lake in production without CISO approval
- NEVER modify subscriber access that could disrupt security monitoring
- NEVER change data retention policies without compliance team review
- NEVER remove log sources that are required for regulatory compliance

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
  - "NEVER suggest disabling security standards"
  - "NEVER suppress findings without investigation"
  - "NEVER remove security integrations"
