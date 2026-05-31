---
title: "E1 — Athena Integration Issues"
description: "Diagnose Athena query integration with Security Lake"
status: active
severity: HIGH
triggers:
  - "Athena query failed"
  - "cannot query security lake"
  - "table not found"
  - "Athena access denied"
owner: devops-agent
objective: "Resolve Athena query integration issues"
context: "Security Lake data is queried via Athena using Glue Data Catalog tables. Queries require Lake Formation permissions, correct database/table names, and Athena workgroup configuration."
---

## Phase 1 — Triage

MUST:
- Verify Glue database exists: `aws glue get-database --name amazon_security_lake_glue_db_<region>`
- List tables: `aws glue get-tables --database-name amazon_security_lake_glue_db_<region>`
- Check Lake Formation permissions for querying principal
- Test simple query: `SELECT count(*) FROM amazon_security_lake_table_<region>_cloud_trail_mgmt_2_0`
- Check Athena query execution for errors: `aws athena get-query-execution --query-execution-id <id>`

SHOULD:
- Verify Athena workgroup has result location configured
- Check if partition projection is working (time-based partitions)
- Verify S3 result bucket permissions

## Phase 2 — Remediate

MUST:
- Grant Lake Formation SELECT on database and tables to querying principal
- Use correct table names (include region and source type)
- Configure Athena workgroup with S3 result location

SHOULD:
- Use partition filters in queries for performance (time range)
- Set up named queries for common Security Lake queries

## Output Format

```yaml
root_cause: "athena-integration — <specific_cause>"
evidence:
  - type: query_error
    content: "<Athena error message>"
  - type: lake_formation
    content: "<permission status>"
severity: HIGH
mitigation:
  immediate: "Fix permissions or query syntax"
  long_term: "Create saved queries and workgroup for Security Lake"
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
