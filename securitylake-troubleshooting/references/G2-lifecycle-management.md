---
title: "G2 — Lifecycle and Storage Management"
description: "Diagnose storage tier transitions and cost management issues"
status: active
severity: LOW
triggers:
  - "storage costs high"
  - "tier transition"
  - "Glacier transition"
  - "storage optimization"
owner: devops-agent
objective: "Resolve storage lifecycle and cost optimization issues"
context: "Security Lake supports storage tier transitions (e.g., S3 Standard to S3 Glacier) via lifecycle configuration. Transitions reduce costs for older data. Glacier data requires restore before querying."
---

## Phase 1 — Triage

MUST:
- Check lifecycle configuration: `aws securitylake list-data-lakes`
- Verify transition rules per source type
- Check current storage distribution: `aws s3 ls s3://<bucket>/ --recursive --summarize`
- Verify Glacier data is restored before querying

SHOULD:
- Calculate storage costs per tier
- Check if transition timing aligns with query patterns

## Phase 2 — Remediate

MUST:
- Configure transitions via `update-data-lake` lifecycle settings
- Ensure Athena queries filter to non-Glacier data or restore first
- Plan transition timing based on query frequency

SHOULD:
- Use S3 Intelligent-Tiering for automatic optimization
- Monitor storage costs monthly

## Output Format

```yaml
root_cause: "lifecycle-management — <specific_cause>"
evidence:
  - type: lifecycle_config
    content: "<transition rules>"
  - type: storage_distribution
    content: "<data per tier>"
severity: LOW
mitigation:
  immediate: "Adjust lifecycle transitions"
  long_term: "Implement cost monitoring dashboards"
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
