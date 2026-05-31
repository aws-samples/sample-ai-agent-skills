---
title: "E3 — Aurora PostgreSQL Logical Replication"
description: "Diagnose Aurora PostgreSQL logical replication issues"
status: active
severity: HIGH
triggers:
  - "logical replication"
  - "publication"
  - "subscription"
  - "rds.logical_replication"
  - "WAL"
  - "replication slot"
owner: devops-agent
objective: "Identify and resolve Aurora PostgreSQL logical replication issues"
context: "Aurora PostgreSQL supports native logical replication using publications and subscriptions. Requires setting rds.logical_replication=1 in the cluster parameter group (changes WAL level to logical, requires reboot). Logical replication is used for CDC, cross-cluster replication, and migration."
---

## Phase 1 — Triage

MUST:
- Check if logical replication is enabled:
  ```sql
  SHOW rds.logical_replication;
  SHOW wal_level;
  -- wal_level must be 'logical'
  ```
- Check replication slots:
  ```sql
  SELECT slot_name, plugin, slot_type, active, restart_lsn, confirmed_flush_lsn
  FROM pg_replication_slots;
  ```
- Check publications:
  ```sql
  SELECT * FROM pg_publication;
  SELECT * FROM pg_publication_tables;
  ```
- Check subscriptions (on subscriber):
  ```sql
  SELECT subname, subenabled, subconninfo, subslotname
  FROM pg_subscription;
  SELECT * FROM pg_stat_subscription;
  ```

SHOULD:
- Check replication lag:
  ```sql
  -- On publisher
  SELECT slot_name, pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) AS lag_bytes
  FROM pg_replication_slots;
  ```
- Check WAL retention:
  ```sql
  SELECT name, setting FROM pg_settings WHERE name IN ('max_replication_slots', 'max_wal_senders', 'wal_sender_timeout');
  ```
- Check for inactive replication slots (can cause WAL accumulation):
  ```sql
  SELECT slot_name, active, pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) AS retained_bytes
  FROM pg_replication_slots WHERE NOT active;
  ```

MAY:
- Check cluster parameter group:
  ```
  aws rds describe-db-cluster-parameters --db-cluster-parameter-group-name <cluster-param-group> \
    --query 'Parameters[?ParameterName==`rds.logical_replication`]'
  ```
- Review replication user permissions

## Phase 2 — Remediate

MUST:
- For logical replication not enabled: set `rds.logical_replication = 1` in cluster parameter group and reboot all instances
- For inactive replication slots: drop unused slots to prevent WAL accumulation: `SELECT pg_drop_replication_slot('<slot_name>');`
- For subscription errors: check pg_stat_subscription for error details

SHOULD:
- Monitor replication slot lag to prevent WAL disk usage growth
- Set `max_replication_slots` and `max_wal_senders` appropriately
- Use `wal_sender_timeout` to detect stale connections

MAY:
- Implement monitoring for replication slot lag
- Consider using pglogical extension for more advanced replication features

## Common Issues

- symptoms: "rds.logical_replication is off"
  diagnosis: "Logical replication not enabled in cluster parameter group."
  resolution: "Set rds.logical_replication=1 in cluster parameter group. Reboot all instances."

- symptoms: "Storage growing due to WAL retention"
  diagnosis: "Inactive replication slot preventing WAL cleanup."
  resolution: "Drop inactive replication slots. Monitor slot activity."

- symptoms: "Subscription not replicating"
  diagnosis: "Subscription disabled, connection issue, or publication mismatch."
  resolution: "Check pg_stat_subscription. Verify connectivity. Enable subscription: ALTER SUBSCRIPTION ... ENABLE."

## Safety Ratings
- GREEN: SHOW rds.logical_replication, SHOW wal_level, pg_replication_slots, pg_publication, pg_subscription, pg_stat_subscription queries — read-only inspection
- YELLOW: modify-db-cluster-parameter-group (rds.logical_replication), ALTER SUBSCRIPTION — recoverable but may require reboot
- RED: pg_drop_replication_slot (can cause data loss if slot is needed), delete-db-cluster — destructive operations

## Escalation Conditions
- "Database serves production traffic"
- "Fix requires enabling rds.logical_replication (needs cluster parameter group change and reboot)"
- "Inactive replication slots causing WAL accumulation and storage growth"
- "Fix requires dropping replication slots"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, subscription connection info (subconninfo contains credentials)
- HIGH: replication slot details and WAL positions (reveal replication topology)
- MEDIUM: publication/subscription configuration, WAL retention settings

## Prohibited Actions
- "NEVER suggest deleting a database cluster to fix replication issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted cluster"
- "NEVER suggest modifying production parameter groups without testing"
- "NEVER suggest force-failover in production without confirming application readiness"
- "NEVER suggest dropping active replication slots without confirming downstream impact"
- "NEVER suggest modifying pg_hba.conf directly — use parameter groups"

## Phase 3 — Rollback
- "Restore from snapshot if logical replication parameter change causes issues"
- "Revert parameter group changes (rds.logical_replication) and reboot if needed"
- "If dropping a replication slot was premature, recreate the slot and resync the subscriber"
- "If failover causes issues, wait for DNS propagation (do NOT failover again immediately)"

## Output Format

```yaml
root_cause: "logical_replication — <specific_cause>"
evidence:
  - type: replication_config
    content: "<rds.logical_replication, wal_level settings>"
  - type: replication_slots
    content: "<slot status and lag>"
  - type: subscription_status
    content: "<pg_stat_subscription output>"
severity: HIGH
mitigation:
  immediate: "Fix replication configuration or drop inactive slots"
  long_term: "Monitor replication slots and implement WAL retention alerting"
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
