---
title: "F2 — Rollup Region Issues"
description: "Diagnose rollup region configuration and data aggregation problems"
status: active
severity: MEDIUM
triggers:
  - "rollup region not working"
  - "data not aggregating"
  - "rollup sync failed"
  - "centralized query"
owner: devops-agent
objective: "Resolve rollup region configuration and sync issues"
context: "Rollup regions aggregate data from contributing regions by copying. Both regions retain data. Rollup enables centralized querying. Configuration issues cause missing data in the rollup region."
---

## Phase 1 — Triage

MUST:
- Check lake configuration for rollup: `aws securitylake list-data-lakes`
- Verify rollup region is configured with contributing regions
- Check for replication exceptions: `aws securitylake list-data-lake-exceptions --regions <rollup-region>`
- Verify data exists in contributing regions first

SHOULD:
- Check S3 replication status for the Security Lake bucket
- Verify KMS key access across regions if using SSE-KMS
- Allow time for cross-region replication (may take hours for initial sync)

## Phase 2 — Remediate

MUST:
- Configure rollup via `update-data-lake` with replication configuration
- Ensure KMS keys in both regions allow cross-region access
- Verify contributing regions have active data

SHOULD:
- Monitor replication lag
- Plan for double storage costs (source + rollup)

## Output Format

```yaml
root_cause: "rollup-region — <specific_cause>"
evidence:
  - type: rollup_config
    content: "<rollup region settings>"
  - type: replication_status
    content: "<sync state>"
severity: MEDIUM
mitigation:
  immediate: "Fix rollup configuration or KMS access"
  long_term: "Monitor replication health continuously"
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
