---
title: "G1 — Data Retention Issues"
description: "Diagnose data retention and expiration policy problems"
status: active
severity: MEDIUM
triggers:
  - "data retention not working"
  - "data not expiring"
  - "retention policy"
  - "storage growing"
owner: devops-agent
objective: "Resolve data retention and lifecycle policy issues"
context: "Security Lake retention is configured per source and region via the update-data-lake API. Retention is implemented as S3 lifecycle rules managed by Security Lake. Do not modify lifecycle rules directly."
---

## Phase 1 — Triage

MUST:
- Check retention configuration: `aws securitylake list-data-lakes --query 'dataLakes[].lifecycleConfiguration'`
- Verify retention is set per source type
- Check S3 lifecycle rules (read-only): `aws s3api get-bucket-lifecycle-configuration --bucket <security-lake-bucket>`
- Verify data age matches retention expectations

SHOULD:
- Check if retention was recently changed (takes time to apply)
- Verify no manual lifecycle rules conflict with managed rules

## Phase 2 — Remediate

MUST:
- Configure retention via `aws securitylake update-data-lake` with lifecycle configuration
- Set retention per source type and region
- Do NOT modify S3 lifecycle rules directly

SHOULD:
- Set different retention for different source types based on compliance needs
- Monitor storage costs after retention changes

## Output Format

```yaml
root_cause: "data-retention — <specific_cause>"
evidence:
  - type: retention_config
    content: "<configured retention periods>"
  - type: lifecycle_rules
    content: "<S3 lifecycle status>"
severity: MEDIUM
mitigation:
  immediate: "Update retention via Security Lake API"
  long_term: "Align retention with compliance requirements"
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
