---
title: "C1 — Target Endpoint Errors"
description: "Diagnose target endpoint connectivity and configuration errors"
status: active
severity: HIGH
triggers:
  - "target connection failed"
  - "cannot write to target"
  - "target endpoint error"
  - "target unreachable"
owner: devops-agent
objective: "Restore connectivity and correct configuration for the target endpoint"
context: "Target endpoint errors prevent DMS from writing migrated data. Causes include connectivity failures, insufficient permissions on the target database, incorrect endpoint configuration, or target-specific limitations."
---

## Phase 1 — Triage

MUST:
- Test target connection: `aws dms test-connection --replication-instance-arn <instance-arn> --endpoint-arn <target-endpoint-arn>`
- Check target endpoint config: `aws dms describe-endpoints --filters Name=endpoint-id,Values=<endpoint-id> --query 'Endpoints[*].{Server:ServerName,Port:Port,Engine:EngineName,Database:DatabaseName,SSL:SslMode}'`
- Check task error messages: `aws dms describe-replication-tasks --filters Name=replication-task-id,Values=<task-id> --query 'ReplicationTasks[*].LastFailureMessage'`
- Check security groups on replication instance: `aws ec2 describe-security-groups --group-ids <sg-id>`

SHOULD:
- Verify target database user has CREATE, INSERT, UPDATE, DELETE permissions
- Check if target database/schema exists
- Verify target is accepting connections (not in maintenance)

MAY:
- Check target database connection limits
- Verify SSL certificate configuration for the target

## Phase 2 — Remediate

MUST:
- Fix network connectivity (security groups, routing, DNS)
- Correct target credentials and permissions
- Ensure target database and schema exist

SHOULD:
- Re-test connection after fixes: `aws dms test-connection --replication-instance-arn <instance-arn> --endpoint-arn <target-endpoint-arn>`
- Grant minimum required permissions to the DMS target user

MAY:
- Configure connection pooling on the target for high-throughput migrations
- Set up target-specific extra connection attributes

## Common Issues

- symptoms: "Target connection timeout"
  diagnosis: "Security group or network path blocking access to target."
  resolution: "Allow inbound on target port from replication instance security group."

- symptoms: "Permission denied on target"
  diagnosis: "DMS user lacks required privileges on target database."
  resolution: "Grant CREATE TABLE, INSERT, UPDATE, DELETE to the DMS user on the target."

- symptoms: "Target database does not exist"
  diagnosis: "Endpoint configured with wrong database name."
  resolution: "Create the target database or correct the endpoint configuration."

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Fix network connectivity (SGs, routing, DNS) | YELLOW | Network change — verify no unintended access |
| Correct target credentials and permissions | YELLOW | Credential/permission change — verify auth works |
| Ensure target database and schema exist | GREEN | Verification — read-only |
| Re-test connection after fixes | GREEN | Verification — non-destructive |
| Grant minimum required permissions to DMS user | YELLOW | Permission change on target database |

## Escalation Conditions

- Task replicates production database
- Fix requires restarting full load
- Target connectivity loss causes growing data sync gap
- Target database serves live traffic and permission changes affect applications

## Data Sensitivity

- **Classification: HIGH**
- Target endpoint configuration contains server addresses, credentials, and database names
- Connection test results reveal network topology between DMS and target
- Security group rules expose network access boundaries
- Target database permissions reveal data access control

## Prohibited Actions

- **NEVER** suggest deleting a replication task without confirming target data is current
- **NEVER** suggest modifying source endpoint during active CDC
- **NEVER** suggest granting DMS user superuser privileges on the target database
- **NEVER** suggest disabling SSL/TLS on target connections

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Modified security group rules | Revert security group rules to previous configuration |
| Updated target credentials | Revert to previous credentials if new ones fail |
| Granted target database permissions | Revoke added permissions after migration completes |

## Output Format

```yaml
root_cause: "target_endpoint — <specific_cause>"
evidence:
  - type: connection_test
    content: "<test connection result>"
  - type: endpoint_config
    content: "<target endpoint configuration>"
severity: HIGH
mitigation:
  immediate: "Fix target connectivity or permissions"
  long_term: "Validate target prerequisites before starting migration"
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
  - "NEVER suggest disabling SSL for replication endpoints"
  - "NEVER suggest public replication instances"
  - "NEVER suggest deleting replication tasks without data verification"
