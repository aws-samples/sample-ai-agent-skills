---
title: "E2 — Point-in-Time Recovery"
description: "Diagnose Neptune PITR issues"
status: active
severity: HIGH
triggers:
  - "point in time recovery"
  - "pitr"
  - "restore to time"
  - "recover data"
owner: devops-agent
objective: "Identify and resolve Neptune point-in-time recovery issues"
context: "Neptune supports continuous backup with point-in-time recovery. PITR creates a new cluster restored to a specific time within the backup retention period."
---

## Phase 1 — Triage

MUST:
- Check backup retention period: `aws neptune describe-db-clusters --db-cluster-identifier <cluster-id> --query 'DBClusters[0].{BackupRetentionPeriod:BackupRetentionPeriod,EarliestRestorableTime:EarliestRestorableTime,LatestRestorableTime:LatestRestorableTime}'`
- Verify the target restore time is within the restorable window
- Check cluster status: `aws neptune describe-db-clusters --db-cluster-identifier <cluster-id> --query 'DBClusters[0].Status'`
- Verify subnet group and security group for restored cluster

SHOULD:
- Check CloudTrail for restore errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=RestoreDBClusterToPointInTime --max-results 10`
- Verify KMS key availability for encrypted clusters
- Ensure sufficient service quotas for new cluster

MAY:
- Test PITR in non-production environment first
- Document the exact restore time needed
- Plan for DNS/endpoint updates after restore

## Phase 2 — Remediate

MUST:
- Restore to a new cluster (PITR always creates a new cluster)
- Configure instances on the restored cluster
- Update application endpoints to point to the new cluster

SHOULD:
- Verify data integrity on restored cluster
- Apply any parameter group customizations to restored cluster
- Update security groups and IAM policies for new cluster

MAY:
- Delete the original cluster after verification
- Update CloudFormation/Terraform to reflect new cluster

## Common Issues

- symptoms: "Restore time is outside the restorable window"
  diagnosis: "The requested time is before EarliestRestorableTime or after LatestRestorableTime."
  resolution: "Choose a time within the restorable window. Increase backup retention period for future."

- symptoms: "PITR restore fails with parameter group error"
  diagnosis: "Custom parameter group not compatible with restored engine version."
  resolution: "Use default parameter group or create compatible one for the restore."

- symptoms: "Restored cluster has no instances"
  diagnosis: "PITR restores the cluster only — instances must be added separately."
  resolution: "Create DB instances on the restored cluster after PITR completes."

## Output Format

```yaml
root_cause: "pitr — <specific_cause>"
evidence:
  - type: backup_window
    content: "<restorable time window>"
  - type: restore_attempt
    content: "<restore operation details>"
severity: HIGH
mitigation:
  immediate: "Complete PITR and configure restored cluster"
  long_term: "Set appropriate backup retention and test PITR regularly"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: MEDIUM — PITR restores contain full database state
- Service Disruption Risk: LOW — PITR operations create new clusters
- Compliance Impact: HIGH — PITR capability is critical for disaster recovery compliance

## Escalation Conditions
- PITR restore fails with internal errors
- PITR window does not cover the required recovery point
- Restored cluster has data inconsistencies
- PITR backup retention period is insufficient for compliance requirements
- PITR restore takes longer than RTO requirements

## Data Sensitivity
- Backup window configurations — INTERNAL
- Recovery point timestamps — INTERNAL
- Restored cluster data — CONFIDENTIAL
- Backup retention settings — INTERNAL
- Never expose recovery point details in external communications

## Prohibited Actions
- Do NOT reduce backup retention period below compliance requirements
- Do NOT rely solely on PITR without also maintaining manual snapshots
- Do NOT restore to a point in time without verifying the target timestamp
- Do NOT delete the source cluster before verifying PITR restore success
- Do NOT skip data validation after PITR restore
- Do NOT expose restored clusters without proper security configurations

## Phase 3 — Rollback
- If PITR restore produces incorrect data, try a different recovery point
- Delete incorrectly restored clusters to avoid confusion
- Restore backup retention settings if modified
- Verify data integrity in the restored cluster before switching traffic
- Document the recovery point used and validation steps performed
- Update disaster recovery procedures based on PITR experience

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
  - "NEVER suggest disabling IAM authentication"
  - "NEVER suggest public access"
  - "NEVER suggest disabling encryption at rest"
