---
title: "D1 — OCSF Schema Mapping Issues"
description: "Diagnose OCSF schema normalization and mapping problems"
status: active
severity: MEDIUM
triggers:
  - "OCSF mapping error"
  - "schema mismatch"
  - "normalization failed"
  - "event class wrong"
owner: devops-agent
objective: "Resolve OCSF schema mapping and normalization issues"
context: "Security Lake normalizes all data to OCSF v1.1+. Native AWS sources are auto-normalized. Custom sources must deliver OCSF-compliant data. Schema issues cause query failures or missing fields."
---

## Phase 1 — Triage

MUST:
- Check Glue table schema: `aws glue get-table --database-name amazon_security_lake_glue_db_<region> --name <table-name>`
- Query sample data: `SELECT * FROM <table> LIMIT 5` via Athena
- Verify OCSF event class matches data type (e.g., class_uid 3001 for DNS Activity)
- Check for NULL fields that should have values

SHOULD:
- Compare custom source schema against OCSF specification
- Verify Parquet column names match OCSF field names exactly
- Check data types match OCSF definitions (string, int, struct)

## Phase 2 — Remediate

MUST:
- Align custom source data with correct OCSF event class
- Ensure all required OCSF fields are populated
- Use correct OCSF data types in Parquet schema

SHOULD:
- Reference OCSF schema documentation for field definitions
- Test with OCSF validation tools before delivery

## Output Format

```yaml
root_cause: "ocsf-mapping — <specific_cause>"
evidence:
  - type: glue_schema
    content: "<table schema>"
  - type: sample_data
    content: "<query results showing issues>"
severity: MEDIUM
mitigation:
  immediate: "Fix schema mapping in custom source"
  long_term: "Implement OCSF validation in data pipeline"
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
