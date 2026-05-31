---
title: "D1 — RDS Instance Action Errors"
description: "Diagnose RDS instance stop budget action errors"
status: active
severity: CRITICAL
triggers: ["rds stop action", "rds instance stop failed", "rds budget action", "database stop action"]
owner: devops-agent
objective: "Resolve RDS instance stop action errors"
context: "RDS stop actions stop running DB instances when budget thresholds are breached. RDS instances auto-restart after 7 days. Aurora clusters cannot be stopped via budget actions."
---

## Phase 1 — Triage

MUST:
- Check action configuration: `aws budgets describe-budget-action --account-id <account-id> --budget-name <name> --action-id <action-id>`
- Verify execution role has rds:StopDBInstance permission: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names rds:StopDBInstance`
- Check target RDS instances: `aws rds describe-db-instances --query 'DBInstances[*].{Id:DBInstanceIdentifier,Status:DBInstanceStatus,Engine:Engine}'`

SHOULD:
- Review action history: `aws budgets describe-budget-action-histories --account-id <account-id> --budget-name <name> --action-id <action-id>`
- Check if instances are part of Multi-AZ or read replica configurations

MAY:
- Verify RDS instances are not Aurora (Aurora clusters have different stop behavior)
- Check for pending maintenance that may prevent stopping

## Phase 2 — Remediate

MUST:
- Grant rds:StopDBInstance to the execution role
- Verify target instances are stoppable (not Aurora Serverless, not part of a cluster)
- Ensure database backups are current before allowing stop actions

SHOULD:
- Set up approval workflow for RDS stop actions
- Configure application connection pooling to handle database unavailability

MAY:
- Implement automated database snapshot before stop
- Set up CloudWatch alarms for RDS instance state changes

## Safety Ratings

```
safety_ratings:
  - "budgets describe-budget-action: GREEN — read-only action check"
  - "rds describe-db-instances: GREEN — read-only instance listing"
  - "iam simulate-principal-policy: GREEN — read-only permission test"
  - "rds stop-db-instance: RED — stops database, causes application downtime"
```

## Escalation Conditions

- RDS stop action targets production databases
- Database has active connections and transactions
- Multi-AZ failover triggered by stop action
- Read replicas become inconsistent after primary stop

## Data Sensitivity

- RDS instance details: HIGH — reveals database configuration and endpoints
- Action configuration: MEDIUM — reveals cost control strategy
- Database connection strings: HIGH — contains access information
- Execution role: MEDIUM — reveals permission scope

## Prohibited Actions

- NEVER stop production databases without application team approval
- NEVER stop RDS instances with pending transactions
- NEVER stop Multi-AZ primary without understanding failover behavior
- NEVER rely on RDS auto-restart (7 days) for production recovery

## Phase 3 — Rollback

1. Start stopped RDS instances: `aws rds start-db-instance --db-instance-identifier <id>`
2. Wait for instance to reach available state: `aws rds wait db-instance-available --db-instance-identifier <id>`
3. Verify application connectivity after restart
4. Reset the budget action: `aws budgets execute-budget-action --account-id <id> --budget-name <name> --action-id <id> --execution-type RESET_BUDGET_ACTION`
5. Check read replica sync status if applicable

## Common Issues

- symptoms: "RDS stop action fails with InvalidDBInstanceState"
  diagnosis: "Instance is not in 'available' state (may be backing up, modifying, or already stopped)."
  resolution: "Wait for instance to reach 'available' state. Check for pending modifications."

## Output Format

```yaml
root_cause: "rds_instance_action — <specific_cause>"
evidence:
  - type: rds_action_error
    content: "<error details>"
severity: CRITICAL
mitigation:
  immediate: "Fix RDS action and restart database"
  long_term: "Implement approval workflows and backup automation"
```

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
