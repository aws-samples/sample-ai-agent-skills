---
title: "E2 — Query Failures and Performance"
description: "Diagnose Security Lake query errors and slow performance"
status: active
severity: MEDIUM
triggers:
  - "query timeout"
  - "slow query"
  - "query returns no data"
  - "scan too much data"
owner: devops-agent
objective: "Resolve query failures and optimize query performance"
context: "Security Lake data is partitioned by region, account, and time. Efficient queries use partition filters. Large time ranges without filters scan excessive data. OCSF schema enables consistent query patterns."
---

## Phase 1 — Triage

MUST:
- Check query execution details: `aws athena get-query-execution --query-execution-id <id>`
- Review data scanned vs returned (cost indicator)
- Verify time range filter uses partition columns (accountid, region, eventday)
- Check if table has data for the queried time range

SHOULD:
- Test with narrow time range first
- Verify column names match OCSF schema (case-sensitive)
- Check for Athena engine version compatibility

## Phase 2 — Remediate

MUST:
- Add partition filters: `WHERE eventday >= '20240101' AND region = 'us-east-1'`
- Use correct OCSF column names in queries
- Limit result sets for exploratory queries

SHOULD:
- Create views for common query patterns
- Use Athena workgroup data scan limits
- Consider using Athena prepared statements

## Output Format

```yaml
root_cause: "query-failure — <specific_cause>"
evidence:
  - type: query_stats
    content: "<data scanned, execution time>"
  - type: error_message
    content: "<query error if any>"
severity: MEDIUM
mitigation:
  immediate: "Fix query with partition filters"
  long_term: "Create optimized saved queries"
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
