---
title: "H2 — Backtrack (Aurora MySQL Only)"
description: "Diagnose Aurora MySQL backtrack issues"
status: active
severity: HIGH
triggers:
  - "backtrack"
  - "rewind"
  - "BacktrackWindowActual"
  - "backtrack window"
  - "point in time rewind"
owner: devops-agent
objective: "Identify and resolve Aurora MySQL backtrack issues"
context: "Aurora backtrack is available ONLY on Aurora MySQL. It rewinds the cluster to a specific point in time without creating a new cluster. Backtrack window is configurable (up to 72 hours). Backtrack affects the entire cluster (all instances). It is NOT available on Aurora PostgreSQL. Backtrack must be enabled at cluster creation."
---

## Phase 1 — Triage

MUST:
- Verify backtrack is enabled on the cluster:
  ```
  aws rds describe-db-clusters --db-cluster-identifier <cluster-id> \
    --query 'DBClusters[0].{BacktrackWindow:BacktrackWindow,BacktrackConsumedChangeRecords:BacktrackConsumedChangeRecords,EarliestBacktrackTime:EarliestBacktrackTime}'
  ```
- Check actual backtrack window:
  ```
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name BacktrackWindowActual \
    --dimensions Name=DBClusterIdentifier,Value=<cluster-id> \
    --start-time <start> --end-time <end> --period 3600 --statistics Average
  ```
- Check backtrack change records:
  ```
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name BacktrackChangeRecordsCreationRate \
    --dimensions Name=DBClusterIdentifier,Value=<cluster-id> ...
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name BacktrackChangeRecordsStored ...
  ```
- Verify the cluster engine is Aurora MySQL (backtrack is NOT available on PostgreSQL):
  ```
  aws rds describe-db-clusters --db-cluster-identifier <cluster-id> --query 'DBClusters[0].Engine'
  ```

SHOULD:
- Check if the target backtrack time is within the actual backtrack window
- Check cluster status before backtracking: cluster must be in available state
- Verify no pending modifications on the cluster

MAY:
- Check the cost of backtrack change records (billed per million records)
- Review if backtrack window is sufficient for the use case

## Phase 2 — Remediate

MUST:
- To backtrack:
  ```
  aws rds backtrack-db-cluster --db-cluster-identifier <cluster-id> \
    --backtrack-to <ISO-8601-timestamp>
  ```
- Understand that backtrack affects ALL instances in the cluster (writer and all readers)
- Backtrack is a destructive operation — all changes after the backtrack point are lost

SHOULD:
- Take a manual snapshot before backtracking as a safety net
- Verify the target time is within EarliestBacktrackTime
- Notify all application teams before backtracking (causes brief unavailability)

MAY:
- Increase backtrack window if current window is insufficient:
  ```
  aws rds modify-db-cluster --db-cluster-identifier <cluster-id> --backtrack-window <seconds>
  ```
- Use backtrack for testing: backtrack, test, then backtrack forward

## Common Issues

- symptoms: "Backtrack not available on this cluster"
  diagnosis: "Backtrack must be enabled at cluster creation. Cannot be enabled later."
  resolution: "Create a new cluster from a snapshot with backtrack enabled."

- symptoms: "Target time is outside the backtrack window"
  diagnosis: "BacktrackWindowActual is shorter than the configured window due to high change rate."
  resolution: "Increase backtrack window. Reduce write rate. Use PITR instead."

- symptoms: "Attempting backtrack on Aurora PostgreSQL"
  diagnosis: "Backtrack is Aurora MySQL only. Not available on PostgreSQL."
  resolution: "Use point-in-time restore (creates a new cluster) for Aurora PostgreSQL."

## Safety Ratings
- GREEN: describe-db-clusters (backtrack config), CloudWatch BacktrackWindowActual/BacktrackChangeRecords metrics — read-only inspection
- YELLOW: modify-db-cluster (backtrack window) — recoverable configuration change
- RED: backtrack-db-cluster — destructive operation that discards all changes after the backtrack point

## Escalation Conditions
- "Database serves production traffic"
- "Backtrack will discard committed transactions (data loss by design)"
- "Fix requires backtracking a production cluster"
- "Backtrack window insufficient for the required recovery point"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, backtrack timestamps (reveal recovery operations)
- MEDIUM: CloudWatch backtrack metrics, change record rates, backtrack window configuration

## Prohibited Actions
- "NEVER suggest backtracking a production cluster without taking a manual snapshot first"
- "NEVER suggest deleting a database cluster to fix backtrack issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted cluster"
- "NEVER suggest backtrack on Aurora PostgreSQL — it is only available on Aurora MySQL"

## Phase 3 — Rollback
- "Take a manual snapshot BEFORE backtracking as a safety net"
- "If backtrack goes too far back, backtrack forward to a more recent point"
- "If backtrack causes unexpected data loss, restore from the pre-backtrack snapshot"
- "Backtrack is NOT reversible without a prior snapshot"

## Output Format

```yaml
root_cause: "backtrack_issue — <specific_cause>"
evidence:
  - type: backtrack_config
    content: "<backtrack window, earliest backtrack time>"
  - type: cloudwatch
    content: "<BacktrackWindowActual, change records metrics>"
  - type: engine
    content: "<cluster engine type>"
severity: HIGH
mitigation:
  immediate: "Perform backtrack or use PITR as alternative"
  long_term: "Monitor backtrack window and change record rate"
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
  - "NEVER suggest making clusters publicly accessible"
  - "NEVER suggest disabling encryption"
  - "NEVER force failover without understanding impact"
