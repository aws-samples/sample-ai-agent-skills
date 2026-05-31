---
title: "G1 — Data Sharing Producer/Consumer Issues"
description: "Diagnose and resolve Redshift data sharing problems between producer and consumer"
status: active
severity: MEDIUM
triggers:
  - "data sharing"
  - "datashare"
  - "producer"
  - "consumer"
  - "shared data"
owner: devops-agent
objective: "Fix data sharing configuration between producer and consumer clusters"
context: "Redshift data sharing allows sharing live data between clusters without copying. The producer cluster creates a datashare and adds schemas/tables. The consumer cluster creates a database from the datashare. Data sharing works across clusters in the same account and region, or cross-account."
---

## Phase 1 — Triage

MUST:
- Check datashare status on producer: `aws redshift describe-data-shares --data-share-arn <share-arn>`
- Check datashare associations: `aws redshift describe-data-share-associations --data-share-arn <share-arn>`
- Check consumer database: `aws redshift-data execute-statement --cluster-identifier <consumer-cluster> --database <db> --db-user <user> --sql "SELECT * FROM svv_datashares"`
- Verify producer cluster is RA3 (data sharing requires RA3 or Serverless): `aws redshift describe-clusters --cluster-identifier <producer-cluster> --query 'Clusters[0].NodeType'`

SHOULD:
- Check datashare objects (schemas, tables added to share)
- Verify consumer has accepted the datashare association
- Check for schema changes on producer that may affect consumer

MAY:
- Check CloudTrail for datashare API events
- Verify cross-account authorization if applicable
- Check consumer query performance on shared data

## Phase 2 — Remediate

MUST:
- If datashare not visible to consumer: verify association is ACTIVE
- If consumer cannot query shared tables: check that tables are added to the datashare on producer
- If node type is DC2: migrate to RA3 — data sharing requires RA3 or Serverless
- If cross-account: ensure authorization is granted on both sides

SHOULD:
- Add entire schemas to datashares instead of individual tables
- Monitor datashare usage metrics
- Document datashare dependencies between clusters

MAY:
- Set up alerts for datashare status changes
- Implement datashare naming conventions
- Use Serverless for consumer workloads on shared data

## Common Issues

- symptoms: "Consumer cannot see shared tables"
  diagnosis: "Tables not added to datashare, or consumer database not created from datashare."
  resolution: "On producer: ALTER DATASHARE ADD TABLE. On consumer: CREATE DATABASE FROM DATASHARE."

- symptoms: "Data sharing not available on DC2 cluster"
  diagnosis: "Data sharing requires RA3 node types or Serverless."
  resolution: "Migrate to RA3 nodes or use Serverless for data sharing."

## Output Format

```yaml
root_cause: "Data sharing — <specific_cause>"
evidence:
  - type: datashare_status
    content: "<describe-data-shares output>"
severity: MEDIUM
mitigation:
  immediate: "Fix datashare configuration or association"
  long_term: "Use RA3/Serverless, document datashare dependencies, monitor status"
```


## Safety Ratings
```
safety_ratings:
  - "Describe datashare status: GREEN — read-only API call"
  - "Check datashare associations: GREEN — read-only API call"
  - "Check consumer database: GREEN — read-only query"
  - "Add tables to datashare: GREEN — metadata-only, no data movement"
  - "Create consumer database from datashare: GREEN — creates read-only database reference"
  - "Remove tables from datashare: YELLOW — breaks consumer queries on those tables"
  - "Drop datashare: RED — breaks all consumer access to shared data"
```

## Escalation Conditions
- Cluster serves production analytics
- Data sharing failure affecting downstream consumer analytics
- Schema changes on producer breaking consumer queries
- Migration from DC2 to RA3 required for data sharing support
- Cross-account sharing authorization needed

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "Datashare configuration: shared table and schema details"
    - "Consumer cluster identifiers: downstream system information"
    - "Shared data: production analytics data accessible to consumers"
  handling: "Datashare configurations reveal data sharing relationships. Restrict access to authorized teams."
```

## Prohibited Actions
- NEVER suggest deleting a cluster without snapshot
- NEVER suggest VACUUM FULL on production during peak hours
- NEVER drop a datashare without confirming all consumers have been notified
- NEVER remove tables from a datashare without verifying consumer query dependencies
- NEVER share sensitive tables without appropriate access controls on the consumer side

## Phase 3 — Rollback
- If tables were added to datashare: remove tables from datashare via ALTER DATASHARE REMOVE TABLE
- If tables were removed from datashare: re-add tables via ALTER DATASHARE ADD TABLE
- If consumer database was created: drop the consumer database if not needed
- If datashare was dropped: recreate the datashare and re-add schemas/tables

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest making clusters publicly accessible"
  - "NEVER suggest disabling audit logging"
  - "NEVER suggest disabling encryption"
