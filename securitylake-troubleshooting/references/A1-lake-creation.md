---
title: "A1 — Lake Creation Failures"
description: "Diagnose Security Lake creation and initialization issues"
status: active
severity: HIGH
triggers:
  - "lake creation failed"
  - "cannot create security lake"
  - "initialization error"
owner: devops-agent
objective: "Identify and resolve Security Lake creation failures"
context: "Security Lake creation provisions S3 buckets, Glue databases, and IAM roles. Failures typically involve IAM permissions, Organizations setup, or service-linked role issues."
---

## Phase 1 — Triage

MUST:
- Check lake status: `aws securitylake list-data-lakes`
- Check for exceptions: `aws securitylake list-data-lake-exceptions --regions <region>`
- Verify service-linked role exists: `aws iam get-role --role-name AWSServiceRoleForSecurityLake`
- Check CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateDataLake --max-results 5`

SHOULD:
- Verify Organizations is set up if using delegated admin
- Check if another account is already delegated admin
- Verify region is supported for Security Lake

## Phase 2 — Remediate

MUST:
- Ensure caller has `securitylake:CreateDataLake` permission
- Allow service-linked role creation: `iam:CreateServiceLinkedRole`
- For Organizations: designate delegated admin first

SHOULD:
- Enable Security Lake in one region first, then expand
- Monitor creation status until COMPLETED

## Output Format

```yaml
root_cause: "lake-creation — <specific_cause>"
evidence:
  - type: lake_status
    content: "<creation state>"
  - type: exceptions
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix permissions or Organizations setup"
  long_term: "Automate lake creation with CloudFormation"
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
