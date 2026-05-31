---
title: "C2 — Subscriber Access Issues"
description: "Diagnose subscriber data access and query problems"
status: active
severity: HIGH
triggers:
  - "subscriber cannot query"
  - "access denied subscriber"
  - "no data for subscriber"
  - "subscriber permission"
owner: devops-agent
objective: "Resolve subscriber data access failures"
context: "Query subscribers need Lake Formation permissions AND IAM permissions. Data access subscribers need SQS/EventBridge notifications working. Access failures are usually permission-related."
---

## Phase 1 — Triage

MUST:
- Check subscriber status: `aws securitylake get-subscriber --subscriber-id <id>`
- For query subscribers: check Lake Formation permissions: `aws lakeformation list-permissions --principal DataLakePrincipal={DataLakePrincipalIdentifier=<role-arn>}`
- Verify subscriber IAM role has Athena, Glue, and S3 permissions
- Test query from subscriber account: `aws athena start-query-execution --query-string "SELECT count(*) FROM <table>" ...`

SHOULD:
- Check if subscriber sources match available data
- Verify Glue database and tables are accessible
- For S3 subscribers: check SQS queue for notifications

## Phase 2 — Remediate

MUST:
- Grant Lake Formation permissions on database and tables to subscriber principal
- Ensure subscriber IAM role has `athena:StartQueryExecution`, `glue:GetTable`, `s3:GetObject`
- Verify S3 bucket allows subscriber role access via Lake Formation

SHOULD:
- Test with simple SELECT query first
- Check Athena query execution history for error details

## Output Format

```yaml
root_cause: "subscriber-access — <specific_cause>"
evidence:
  - type: lake_formation_grants
    content: "<granted permissions>"
  - type: iam_permissions
    content: "<IAM policy evaluation>"
severity: HIGH
mitigation:
  immediate: "Fix Lake Formation and IAM permissions"
  long_term: "Automate subscriber permission management"
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
