---
title: "A2 — Instance Issues"
description: "Diagnose DocumentDB instance health and configuration problems"
status: active
severity: HIGH
triggers:
  - "instance failed"
  - "instance not available"
  - "instance rebooting"
  - "instance class"
  - "writer failover"
owner: devops-agent
objective: "Identify and resolve DocumentDB instance health and configuration issues"
context: "DocumentDB instances can experience issues from instance class limitations, failover events, maintenance operations, or resource exhaustion (CPU, memory, connections). The writer instance handles all writes while reader instances serve read traffic."
---

## Phase 1 — Triage

MUST:
- Describe cluster instances: `aws docdb describe-db-instances --filters Name=db-cluster-id,Values=<cluster-id>`
- Check instance Status (available, backing-up, modifying, rebooting, failed)
- Check CPU and memory: `aws cloudwatch get-metric-statistics --namespace AWS/DocDB --metric-name CPUUtilization --dimensions Name=DBInstanceIdentifier,Value=<instance-id> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Check FreeableMemory: `aws cloudwatch get-metric-statistics --namespace AWS/DocDB --metric-name FreeableMemory --dimensions Name=DBInstanceIdentifier,Value=<instance-id> --start-time <start> --end-time <end> --period 300 --statistics Minimum`

SHOULD:
- Check instance events: `aws docdb describe-events --source-identifier <instance-id> --source-type db-instance --duration 1440`
- Verify instance class is appropriate for workload
- Check DatabaseConnections metric against instance limits
- Review pending maintenance actions: `aws docdb describe-pending-maintenance-actions`

MAY:
- Compare performance across instances in the cluster
- Check for instance class availability in the AZ
- Review instance modification history via CloudTrail

## Phase 2 — Remediate

MUST:
- Address resource exhaustion (scale up instance class if CPU/memory constrained)
- Resolve any failed instance state
- Ensure writer instance is healthy for write operations

SHOULD:
- Right-size instance classes based on workload metrics
- Add reader instances for read-heavy workloads
- Schedule maintenance during low-traffic windows

MAY:
- Implement auto-scaling for reader instances
- Set up CloudWatch alarms for instance health metrics

## Common Issues

- symptoms: "Instance CPU consistently above 80%"
  diagnosis: "Instance class is undersized for the workload or queries are inefficient."
  resolution: "Scale up the instance class or optimize queries/indexes. Check for missing indexes."

- symptoms: "Instance enters rebooting state unexpectedly"
  diagnosis: "Pending maintenance action was applied or OOM condition triggered a restart."
  resolution: "Check pending maintenance actions. Monitor FreeableMemory. Scale up if memory constrained."

- symptoms: "Writer failover causes application errors"
  diagnosis: "Application does not handle failover gracefully — connections to old writer fail."
  resolution: "Use the cluster endpoint for writes (auto-routes to current writer). Implement retry logic."

## Output Format

```yaml
root_cause: "instance_issue — <specific_cause>"
evidence:
  - type: instance
    content: "<describe-db-instances output>"
  - type: metrics
    content: "<CPU, memory, connections metrics>"
  - type: events
    content: "<recent instance events>"
severity: HIGH
mitigation:
  immediate: "Scale instance class or resolve failed state"
  long_term: "Right-size instances and implement monitoring"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | HIGH — Instance changes affect database availability and data access |
| Blast Radius | HIGH — Writer instance issues affect all write operations; scaling causes brief unavailability |
| Rollback Complexity | MEDIUM — Instance type changes require stop/start |
| Compliance Scope | HIGH — Database availability is a compliance requirement |

## Escalation Conditions

- Escalate to operations if writer instance is unhealthy
- Escalate to application teams if failover occurs (endpoint changes)
- Escalate to management if instance scaling is needed during business hours

## Data Sensitivity

- **Instance metrics** reveal workload patterns — operational data
- **Instance events** may contain maintenance details

## PROHIBITED Actions

- Do NOT scale down instance types during peak hours
- Do NOT reboot the writer instance without coordinating with application teams
- Do NOT ignore high CPU or low memory alerts — they indicate impending issues

## Phase 3 — Rollback

- Revert instance type change: `aws docdb modify-db-instance --db-instance-identifier <id> --db-instance-class <previous-class> --apply-immediately`
- Monitor instance health after rollback
- Verify application connectivity after any instance changes

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
