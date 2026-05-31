---
title: "C3 — Connection Pooling Issues"
description: "Diagnose connection pool exhaustion and configuration problems"
status: active
severity: MEDIUM
triggers:
  - "connection pool"
  - "too many connections"
  - "connection exhaustion"
  - "pool timeout"
  - "max connections"
owner: devops-agent
objective: "Identify and resolve connection pooling issues with DocumentDB"
context: "Each DocumentDB instance has a maximum connection limit based on instance class. Connection pooling is essential to avoid exhaustion. Issues include pool misconfiguration, connection leaks, and insufficient pool sizing for the application scale."
---

## Phase 1 — Triage

MUST:
- Check current connections: `aws cloudwatch get-metric-statistics --namespace AWS/DocDB --metric-name DatabaseConnections --dimensions Name=DBInstanceIdentifier,Value=<instance-id> --start-time <start> --end-time <end> --period 60 --statistics Maximum`
- Describe instance for class and connection limits: `aws docdb describe-db-instances --db-instance-identifier <instance-id>`
- Check active connections: `mongo --tls --host <endpoint> --eval "db.serverStatus().connections"`
- Verify application connection pool configuration

SHOULD:
- Check for connection leaks (connections growing without release)
- Verify pool min/max size settings in the application driver
- Check connection idle timeout settings
- Monitor connections per application instance

MAY:
- Review application connection lifecycle (open, use, return to pool)
- Check for Lambda cold start connection accumulation
- Analyze connection distribution across reader instances

## Phase 2 — Remediate

MUST:
- Configure connection pool max size to stay within instance limits
- Fix connection leaks in application code (ensure connections are returned to pool)
- Distribute connections across writer and reader endpoints appropriately

SHOULD:
- Set pool min size to handle baseline load without connection creation latency
- Configure idle connection timeout to release unused connections
- Use reader endpoint for read operations to distribute connection load

MAY:
- Implement connection pool monitoring in the application
- Scale up instance class for higher connection limits
- Add reader instances to increase total cluster connection capacity

## Common Issues

- symptoms: "MongoServerError: too many open connections"
  diagnosis: "Application has exhausted the instance connection limit."
  resolution: "Reduce pool max size per application instance. Total pools × max size must be < instance limit."

- symptoms: "Connection pool timeout — cannot acquire connection"
  diagnosis: "All pool connections are in use and none are being returned."
  resolution: "Check for connection leaks. Increase pool size. Add connection wait timeout."

- symptoms: "Lambda functions accumulate connections over time"
  diagnosis: "Each Lambda cold start creates new connections that are not released between invocations."
  resolution: "Initialize the connection pool outside the handler. Use connection pooling with Lambda context reuse."

## Output Format

```yaml
root_cause: "connection_pooling — <specific_cause>"
evidence:
  - type: connections
    content: "<current and max connection counts>"
  - type: pool_config
    content: "<application pool configuration>"
  - type: metrics
    content: "<DatabaseConnections trend>"
severity: MEDIUM
mitigation:
  immediate: "Fix pool configuration or connection leaks"
  long_term: "Implement connection pool monitoring and right-sizing"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | MEDIUM — Connection pool configuration affects database access |
| Blast Radius | HIGH — Pool exhaustion causes application-wide database failures |
| Rollback Complexity | LOW — Pool settings can be changed in application configuration |
| Compliance Scope | MEDIUM — Connection management affects application availability |

## Escalation Conditions

- Escalate to application teams if connection pool exhaustion causes outages
- Escalate to DBA team if instance connection limits are reached
- Escalate to operations if connection leaks are detected

## Data Sensitivity

- **Connection pool configuration** is application operational data
- **Connection counts** reveal application scale

## PROHIBITED Actions

- Do NOT set pool max size higher than the instance connection limit
- Do NOT deploy connection pool changes without testing
- Do NOT ignore connection leak warnings

## Phase 3 — Rollback

- Revert application connection pool settings to previous configuration
- Restart application instances to reset connection pools
- Monitor DatabaseConnections metric after rollback

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
