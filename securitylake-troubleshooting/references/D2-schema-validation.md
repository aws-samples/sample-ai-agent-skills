---
title: "D2 — Schema Validation Errors"
description: "Diagnose Parquet schema validation and data format issues"
status: active
severity: MEDIUM
triggers:
  - "schema validation failed"
  - "Parquet format error"
  - "data type mismatch"
  - "column not found"
owner: devops-agent
objective: "Resolve schema validation and data format errors"
context: "Security Lake expects Apache Parquet files with OCSF-compliant schema. Schema validation errors prevent data from being queryable. Common issues include wrong column names, data types, and missing required fields."
---

## Phase 1 — Triage

MUST:
- Check Glue table for schema: `aws glue get-table --database-name amazon_security_lake_glue_db_<region> --name <table>`
- Verify Parquet file schema matches Glue table definition
- Check for ingestion exceptions: `aws securitylake list-data-lake-exceptions --regions <region>`
- Test query for schema errors: run Athena query and check error message

SHOULD:
- Use `parquet-tools schema` to inspect delivered Parquet files
- Compare column names case-sensitively (OCSF uses snake_case)
- Verify nested struct types match OCSF definitions

## Phase 2 — Remediate

MUST:
- Fix Parquet schema to match OCSF specification exactly
- Ensure column names are lowercase snake_case
- Use correct Parquet logical types (e.g., TIMESTAMP_MILLIS)

SHOULD:
- Validate schema before delivery using OCSF tools
- Re-deliver corrected data to the custom source prefix

## Output Format

```yaml
root_cause: "schema-validation — <specific_cause>"
evidence:
  - type: schema_comparison
    content: "<expected vs actual schema>"
  - type: validation_error
    content: "<specific error message>"
severity: MEDIUM
mitigation:
  immediate: "Fix Parquet schema to match OCSF"
  long_term: "Add schema validation to delivery pipeline"
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
