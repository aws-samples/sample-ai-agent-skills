---
title: "B2 — Partitioning Issues"
description: "Diagnose and resolve Athena table partitioning problems"
status: active
severity: HIGH
triggers:
  - "partition"
  - "MSCK REPAIR"
  - "partition projection"
  - "empty results"
  - "no data found"
owner: devops-agent
objective: "Identify partitioning issues causing empty results, stale data, or performance problems"
context: "Athena partitions map to S3 prefixes and are tracked in Glue Data Catalog. Missing partitions, wrong projection config, or stale metadata cause empty results."
---

## Phase 1 — Triage

MUST:
- Check partition keys: `aws glue get-table --database-name <db> --name <table> --query 'Table.PartitionKeys'`
- List existing partitions: `aws glue get-partitions --database-name <db> --table-name <table> --max-items 20`
- Check partition projection properties: `aws glue get-table --database-name <db> --name <table> --query 'Table.Parameters'`
- Verify S3 prefix structure matches partition scheme: `aws s3 ls <table-location> --delimiter /`

SHOULD:
- Check if new data has corresponding partitions registered
- Verify partition projection configuration (type, range, format, interval)
- Check for partition count limits (typically 10M per table)

MAY:
- Run MSCK REPAIR TABLE to sync partitions from S3
- Check Glue crawler schedule and last run status: `aws glue get-crawler --name <crawler-name>`

## Phase 2 — Remediate

MUST:
- If partitions missing: run `MSCK REPAIR TABLE <db>.<table>` or `ALTER TABLE ADD PARTITION`
- If projection misconfigured: fix projection.* table properties in Glue
- If S3 structure wrong: ensure S3 prefixes match partition key=value format
- If partition values stale: update via Glue crawler or manual partition add

SHOULD:
- Use partition projection for time-series data to avoid MSCK REPAIR
- Set up Glue crawler on schedule for automatic partition discovery
- Use Hive-compatible S3 paths (key=value format)

MAY:
- Implement Lambda-based partition registration on S3 events
- Use ALTER TABLE ADD PARTITION for individual partition adds

## Common Issues

- symptoms: "Query returns zero rows but data exists in S3"
  diagnosis: "Partitions not registered in Glue Data Catalog."
  resolution: "Run MSCK REPAIR TABLE or add partitions manually with ALTER TABLE ADD PARTITION."

- symptoms: "Partition projection returns empty results for recent dates"
  diagnosis: "Projection range.end is a fixed date that hasn't been updated."
  resolution: "Use NOW as the range end for date projections: 'projection.date.range'='2020-01-01,NOW'."

## Output Format

```yaml
root_cause: "Partitioning issue — <specific_cause>"
evidence:
  - type: partition_metadata
    content: "<partition keys, registered partitions, S3 structure>"
  - type: projection_config
    content: "<projection properties if applicable>"
severity: HIGH
mitigation:
  immediate: "Register missing partitions or fix projection configuration"
  long_term: "Implement automated partition management (projection, crawler, or Lambda)"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Run MSCK REPAIR TABLE | GREEN | Adds missing partitions — non-destructive |
| ALTER TABLE ADD PARTITION | GREEN | Adds single partition — non-destructive |
| Fix partition projection properties | YELLOW | Modifies table properties — affects all queries |
| Ensure S3 prefixes match partition format | YELLOW | May require data reorganization |
| Set up Glue crawler on schedule | GREEN | Automated partition discovery — non-destructive |
| Lambda-based partition registration | GREEN | Automated partition management |

## Escalation Conditions

- Query accesses sensitive data (PII, financial, healthcare records)
- Fix requires modifying workgroup settings that affect multiple teams
- Partition projection changes affect all queries on the table
- S3 data reorganization may affect other applications reading the same data

## Data Sensitivity

- Partition metadata may reveal data organization patterns (dates, regions, customer segments)
- S3 prefix listing may expose sensitive data path structures
- Partition projection configuration may reveal data retention and organization policies

## Prohibited Actions

- NEVER suggest scanning entire tables without LIMIT to debug — this can incur massive costs and expose sensitive data
- NEVER drop partitions without confirming the data should be inaccessible
- NEVER reorganize S3 data without understanding all applications reading from the location

## Phase 3 — Rollback

- If partition projection was misconfigured: remove projection properties using `ALTER TABLE UNSET TBLPROPERTIES`
- If wrong partitions were added: remove with `ALTER TABLE DROP PARTITION`
- If MSCK REPAIR added incorrect partitions: drop the incorrect partitions individually
- If S3 data was reorganized: restore original prefix structure (may require data copy)
- Document all partition and table property changes for audit trail

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
  - "NEVER suggest disabling query result encryption"
  - "NEVER suggest overly broad S3 access for workgroups"
  - "NEVER suggest disabling workgroup enforcement"
