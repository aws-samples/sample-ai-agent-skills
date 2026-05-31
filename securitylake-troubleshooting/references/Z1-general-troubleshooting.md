---
title: "Z1 — General Security Lake Troubleshooting"
description: "Catch-all runbook for Security Lake issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers:
  - "Security Lake not working"
  - "security lake issue"
  - "security lake error"
owner: devops-agent
objective: "Systematic triage of unspecified Security Lake issues"
context: "Use when the user reports a Security Lake problem without specific symptoms. Walk through: lake status, sources, subscribers, schema, queries, permissions."
---

## Phase 1 — Triage

MUST:
- Check lake status: `aws securitylake list-data-lakes`
- Check exceptions: `aws securitylake list-data-lake-exceptions --regions <region>`
- Check sources: `aws securitylake get-data-lake-sources --accounts <account-id>`
- Check subscribers: `aws securitylake list-subscribers`
- Verify Glue tables exist: `aws glue get-tables --database-name amazon_security_lake_glue_db_<region>`
- Test Athena query on a known table
- Check Lake Formation permissions

SHOULD:
- Review CloudTrail for recent changes
- Check CloudWatch metrics for ingestion rates

## Phase 2 — Route to specific runbook

Based on findings, route to:
- A1/A2 for lake setup issues
- B1/B2 for source/custom source issues
- C1/C2 for subscriber issues
- D1/D2 for schema issues
- E1/E2 for query issues
- F1/F2 for cross-account/rollup issues
- G1/G2 for retention issues
- H1/H2 for permission issues

## Output Format

```yaml
root_cause: "general — <routed_to_specific_runbook>"
evidence:
  - type: triage_summary
    content: "<findings from systematic check>"
severity: MEDIUM
mitigation:
  immediate: "Follow specific runbook"
  long_term: "Implement comprehensive monitoring"
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
