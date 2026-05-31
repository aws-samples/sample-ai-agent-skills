---
title: "G2 — Point-in-Time Recovery Issues"
description: "Diagnose PITR configuration and restore problems on DocumentDB"
status: active
severity: HIGH
triggers:
  - "point in time"
  - "PITR"
  - "restore to time"
  - "continuous backup"
  - "restore cluster"
owner: devops-agent
objective: "Identify and resolve point-in-time recovery issues on DocumentDB"
context: "DocumentDB PITR allows restoring to any second within the backup retention period (up to 35 days). PITR creates a new cluster. Issues include restore time selection, new cluster configuration, and data consistency expectations."
---

## Phase 1 — Triage

MUST:
- Check backup retention period: `aws docdb describe-db-clusters --db-cluster-identifier <cluster-id>`
- Verify the desired restore time is within the retention window
- Check LatestRestorableTime and EarliestRestorableTime from cluster description
- Verify subnet group and security group for the new cluster

SHOULD:
- Confirm the restore time is valid (between earliest and latest restorable time)
- Check KMS key availability for the restored cluster
- Verify instance class availability in the target AZ
- Review the restore parameters

MAY:
- Estimate restore duration based on cluster size
- Plan for DNS/endpoint changes (new cluster has new endpoints)
- Verify application can connect to the new cluster

## Phase 2 — Remediate

MUST:
- Use restore-db-cluster-to-point-in-time with the correct restore time
- Create instances in the restored cluster (restore only creates the cluster, not instances)
- Update application connection strings to point to the new cluster

SHOULD:
- Verify data consistency after restore
- Test the restored cluster before switching production traffic
- Document the restore procedure

MAY:
- Automate PITR restore procedures
- Set up monitoring on the restored cluster

## Common Issues

- symptoms: "Restore time is outside the restorable window"
  diagnosis: "The requested restore time is before EarliestRestorableTime or after LatestRestorableTime."
  resolution: "Choose a time within the window. Increase backup retention period for future needs."

- symptoms: "Restored cluster has no instances"
  diagnosis: "PITR restore creates only the cluster — instances must be added separately."
  resolution: "Create instances in the restored cluster: aws docdb create-db-instance --db-cluster-identifier <restored-cluster>."

- symptoms: "Application cannot connect to restored cluster"
  diagnosis: "The restored cluster has new endpoints different from the original."
  resolution: "Update application connection strings with the new cluster endpoint."

## Output Format

```yaml
root_cause: "pitr — <specific_cause>"
evidence:
  - type: cluster
    content: "<backup retention and restorable time window>"
  - type: restore
    content: "<restore operation status>"
  - type: configuration
    content: "<restored cluster configuration>"
severity: HIGH
mitigation:
  immediate: "Complete restore and verify data consistency"
  long_term: "Document PITR procedures and test regularly"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | CRITICAL — PITR restores contain production data |
| Blast Radius | LOW — PITR creates a new cluster; does not affect the original |
| Rollback Complexity | MEDIUM — Restored cluster needs instances and application reconfiguration |
| Compliance Scope | HIGH — PITR is a critical disaster recovery capability |

## Escalation Conditions

- Escalate to data owners before performing PITR — confirm the target restore time
- Escalate to operations if PITR is needed for production data recovery
- Escalate to compliance if data loss occurred and PITR is the recovery mechanism

## Data Sensitivity

- **Restored cluster** contains production data — apply same security controls as original
- **New cluster endpoints** must be secured before routing application traffic
- **Original cluster** remains unaffected by the restore

## PROHIBITED Actions

- Do NOT perform PITR without confirming the target restore time with data owners
- Do NOT route production traffic to the restored cluster without validation
- Do NOT delete the original cluster until the restored cluster is verified
- Do NOT create the restored cluster without encryption matching the original

## Phase 3 — Rollback

- If the restored cluster has incorrect data, delete it and restore to a different point in time
- The original cluster is unaffected — no rollback needed for the original
- Update application connection strings to point back to the original cluster if the restore was unnecessary

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
  - "NEVER suggest disabling TLS"
  - "NEVER suggest making clusters publicly accessible"
  - "NEVER suggest disabling audit logging"
