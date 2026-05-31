---
title: "H1 — IAM Role Issues"
description: "Diagnose IAM role and permission problems for Security Lake"
status: active
severity: HIGH
triggers:
  - "IAM permission denied"
  - "role not authorized"
  - "service-linked role"
  - "assume role failed"
owner: devops-agent
objective: "Resolve IAM role and permission issues"
context: "Security Lake uses service-linked roles for internal operations and IAM roles for subscriber access. The service-linked role (AWSServiceRoleForSecurityLake) is auto-created. Subscriber roles need both IAM and Lake Formation permissions."
---

## Phase 1 — Triage

MUST:
- Check service-linked role: `aws iam get-role --role-name AWSServiceRoleForSecurityLake`
- Verify caller permissions: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names securitylake:ListDataLakes`
- Check subscriber role trust policy allows the subscriber account
- Verify no SCP blocking Security Lake actions

SHOULD:
- Check if service-linked role was accidentally deleted
- Verify KMS key policy allows the service-linked role
- Check CloudTrail for AccessDenied events

## Phase 2 — Remediate

MUST:
- Recreate service-linked role if deleted: `aws iam create-service-linked-role --aws-service-name securitylake.amazonaws.com`
- Ensure admin role has `securitylake:*` permissions
- Configure subscriber roles with correct trust policies

SHOULD:
- Use least-privilege IAM policies
- Audit IAM permissions regularly

## Output Format

```yaml
root_cause: "iam-roles — <specific_cause>"
evidence:
  - type: role_status
    content: "<role existence and policy>"
  - type: permission_check
    content: "<simulation result>"
severity: HIGH
mitigation:
  immediate: "Fix IAM role or permissions"
  long_term: "Implement IAM permission boundaries"
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
