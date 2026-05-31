---
title: "Z1 — General DMS Troubleshooting"
description: "Catch-all systematic investigation for DMS issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers:
  - "DMS issue"
  - "migration problem"
  - "DMS error"
  - "replication not working"
  - "general troubleshooting"
owner: devops-agent
objective: "Systematically investigate DMS issues using a structured diagnostic approach"
context: "This runbook provides a general-purpose investigation framework for DMS issues that don't clearly match a specific runbook. It covers task health, endpoint connectivity, instance resources, and systematic elimination of common causes. Use this as a starting point, then pivot to a specific runbook."
---

## Phase 1 — Triage

MUST:
- Check all replication tasks: `aws dms describe-replication-tasks --query 'ReplicationTasks[*].{Id:ReplicationTaskIdentifier,Status:Status,Type:MigrationType,StopReason:StopReason,LastError:LastFailureMessage}'`
- Check replication instances: `aws dms describe-replication-instances --query 'ReplicationInstances[*].{Id:ReplicationInstanceIdentifier,Class:ReplicationInstanceClass,Status:ReplicationInstanceStatus,Storage:AllocatedStorage,MultiAZ:MultiAZ}'`
- Check all endpoints: `aws dms describe-endpoints --query 'Endpoints[*].{Id:EndpointIdentifier,Type:EndpointType,Engine:EngineName,Status:Status}'`
- Test endpoint connections: `aws dms test-connection --replication-instance-arn <instance-arn> --endpoint-arn <endpoint-arn>`

SHOULD:
- Check CloudWatch metrics for the replication instance (CPU, memory, storage, network)
- Review table statistics for error tables
- Check CloudTrail for recent DMS events
- Review DMS event subscriptions for notifications

MAY:
- Check AWS Health Dashboard for DMS service issues
- Review application logs for migration-related errors
- Check DMS task CloudWatch Logs for detailed diagnostics

## Phase 2 — Remediate

MUST:
- Based on triage findings, pivot to the appropriate specific runbook:
  - Task issues → A1 (failures), A2 (stuck), A3 (CDC lag)
  - Source issues → B1 (connectivity), B2 (engine errors), B3 (logging)
  - Target issues → C1 (errors), C2 (type mapping), C3 (apply errors)
  - Instance issues → D1 (sizing), D2 (storage)
  - Schema issues → E1 (conversion), E2 (table mapping)
  - Performance → F1 (throughput), F2 (latency), F3 (LOB)
  - Validation → G1 (validation), G2 (row count)
  - Security → H1 (IAM), H2 (VPC)
- Document findings and root cause

SHOULD:
- Check for multiple concurrent issues
- Review recent changes that may have caused the issue

MAY:
- Set up comprehensive monitoring to prevent recurrence
- Create a post-incident report for significant issues

## Common Issues

- symptoms: "Migration not working — unclear what's wrong"
  diagnosis: "Multiple possible causes. Follow systematic triage."
  resolution: "Check task status → endpoint connectivity → instance health → table statistics."

- symptoms: "Migration was working but stopped"
  diagnosis: "Recent change to endpoints, instance, or source/target database."
  resolution: "Check CloudTrail for recent changes. Check task LastFailureMessage."

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Check all replication tasks and instances | GREEN | Read-only investigation |
| Test endpoint connections | GREEN | Verification — non-destructive |
| Check CloudWatch metrics and CloudTrail | GREEN | Read-only investigation |
| Pivot to specific runbook | YELLOW | Specific runbook may include state-changing actions |
| Set up comprehensive monitoring | GREEN | Monitoring — non-destructive |

## Escalation Conditions

- Task replicates production database
- Fix requires restarting full load
- Issue cannot be classified after systematic investigation
- Multiple concurrent issues affect replication infrastructure

## Data Sensitivity

- **Classification: HIGH**
- Task status reveals migration state and data sync health
- Endpoint connectivity details expose database connection information
- Instance metrics reveal replication infrastructure sizing
- CloudTrail events expose all DMS API calls and configurations

## Prohibited Actions

- **NEVER** suggest deleting a replication task without confirming target data is current
- **NEVER** suggest modifying source endpoint during active CDC
- **NEVER** suggest making broad configuration changes during general investigation
- **NEVER** suggest restarting replication instances without understanding impact on running tasks

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| General triage is read-only | No rollback needed for Phase 1 investigation |
| Pivot to specific runbook | Follow rollback steps in the specific runbook |

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: task_status
    content: "<task status and errors>"
  - type: endpoint_connectivity
    content: "<endpoint connection test results>"
  - type: instance_health
    content: "<instance metrics>"
severity: MEDIUM
general_analysis:
  task_state: "running | stopped | failed"
  endpoints: "connected | failed"
  instance: "healthy | resource-constrained"
  specific_runbook: "<recommended runbook ID>"
mitigation:
  immediate: "Follow the identified specific runbook"
  long_term: "Implement comprehensive monitoring and alerting"
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
