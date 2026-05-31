---
title: "C2 — HDFS Issues"
description: "Diagnose HDFS storage and replication issues on EMR"
status: active
severity: MEDIUM
triggers:
  - "HDFS full"
  - "HDFS replication"
  - "NameNode issue"
  - "HDFS capacity"
owner: devops-agent
objective: "Identify and resolve HDFS issues on EMR"
context: "HDFS on EMR uses core node local storage and EBS volumes. HDFS capacity is limited by core node count and storage. Replication factor affects usable capacity. HDFS is ephemeral on EMR — data is lost when the cluster terminates. Use S3 for persistent storage."
---

## Phase 1 — Triage

MUST:
- Check HDFS capacity: `hdfs dfsadmin -report` (via SSH or step)
- Verify HDFS replication factor: check dfs.replication in hdfs-site.xml
- Check for under-replicated blocks: `hdfs fsck / -blocks -locations`
- Verify core node count and storage configuration

SHOULD:
- Check HDFS NameNode logs for errors
- Verify EBS volume health on core nodes
- Check for large temporary files consuming HDFS space
- Review HDFS usage by directory

MAY:
- Check HDFS balancer status
- Review HDFS encryption zone configuration

## Phase 2 — Remediate

MUST:
- Clean up unnecessary HDFS data
- Add core nodes if HDFS capacity is insufficient
- Fix under-replicated blocks by ensuring sufficient core nodes

SHOULD:
- Use S3 as primary storage instead of HDFS
- Configure appropriate replication factor (2 for most EMR workloads)
- Monitor HDFS capacity via CloudWatch

MAY:
- Run HDFS balancer for even data distribution
- Implement HDFS cleanup automation

## Common Issues

- symptoms: "HDFS capacity full"
  diagnosis: "Insufficient core node storage for the data volume."
  resolution: "Add core nodes, increase EBS volumes, or move data to S3."

- symptoms: "Under-replicated blocks"
  diagnosis: "Insufficient core nodes for the replication factor."
  resolution: "Add core nodes or reduce replication factor."

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Clean up unnecessary HDFS data | YELLOW | Data deletion — verify data is not needed |
| Add core nodes for HDFS capacity | YELLOW | Cluster scaling — cost increase |
| Reduce replication factor | YELLOW | Reduces data redundancy — increases risk of data loss |
| Run HDFS balancer | GREEN | Redistributes data — non-destructive |
| Move data to S3 | GREEN | Data migration — preserves data |

## Escalation Conditions

- Cluster runs production Spark jobs
- Fix requires cluster termination and recreation
- HDFS contains data not yet persisted to S3
- Under-replicated blocks risk data loss if a node fails

## Data Sensitivity

- **Classification: HIGH**
- HDFS data may contain sensitive intermediate processing results
- HDFS capacity reports reveal data volumes and storage architecture
- Replication configuration exposes data durability strategy
- HDFS encryption zone configuration reveals data classification

## Prohibited Actions

- **NEVER** suggest terminating a cluster with running steps without saving state
- **NEVER** suggest disabling encryption on a cluster or its storage
- **NEVER** suggest deleting HDFS data without confirming it is persisted elsewhere
- **NEVER** suggest reducing replication factor to 1 on production clusters

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Deleted HDFS data | Restore from S3 or other backup if data was needed |
| Added core nodes | Remove added core nodes after decommissioning (if safe) |
| Reduced replication factor | Increase replication factor back to previous value |
| Ran HDFS balancer | Balancer is idempotent — no rollback needed |

## Output Format

```yaml
root_cause: "hdfs — <specific_cause>"
evidence:
  - type: hdfs_report
    content: "<HDFS capacity and health>"
  - type: replication
    content: "<replication status>"
severity: MEDIUM
mitigation:
  immediate: "Fix HDFS capacity or replication"
  long_term: "Use S3 as primary storage, HDFS for temporary data only"
```

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
  - "NEVER suggest disabling encryption"
  - "NEVER suggest public subnet placement for clusters"
  - "NEVER suggest removing Kerberos authentication"
