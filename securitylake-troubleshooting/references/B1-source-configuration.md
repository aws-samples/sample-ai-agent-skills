---
title: "B1 — Source Configuration Issues"
description: "Diagnose AWS log source configuration and ingestion problems"
status: active
severity: HIGH
triggers:
  - "source not ingesting"
  - "no data from source"
  - "source configuration failed"
  - "CloudTrail not appearing"
owner: devops-agent
objective: "Resolve log source configuration and ingestion issues"
context: "Security Lake supports native AWS sources: CloudTrail, VPC Flow Logs, Route 53 Resolver, S3 Data Events, Lambda Data Events, and EKS Audit Logs. Sources must be explicitly enabled per region and account."
---

## Phase 1 — Triage

MUST:
- Check configured sources: `aws securitylake get-data-lake-sources --accounts <account-id>`
- Check for ingestion exceptions: `aws securitylake list-data-lake-exceptions --regions <region>`
- Verify source is enabled for the correct region and account
- Check if source service is active (e.g., CloudTrail is enabled)

SHOULD:
- Verify data appears in Glue tables after ingestion delay (15-30 min)
- Check S3 bucket for source-specific prefixes
- Verify no SCP blocking the source service

## Phase 2 — Remediate

MUST:
- Enable source via `create-aws-log-source` with correct source name and regions
- Ensure underlying service is active (CloudTrail trail exists, VPC Flow Logs enabled)
- Wait for ingestion delay before diagnosing missing data

SHOULD:
- Enable sources in all required regions
- Monitor exceptions after enabling new sources

## Output Format

```yaml
root_cause: "source-configuration — <specific_cause>"
evidence:
  - type: source_status
    content: "<enabled sources>"
  - type: exceptions
    content: "<ingestion errors>"
severity: HIGH
mitigation:
  immediate: "Enable source or fix ingestion errors"
  long_term: "Automate source enablement across accounts"
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
