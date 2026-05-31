---
title: "Z1 — General DocumentDB Troubleshooting"
description: "Catch-all runbook for DocumentDB issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers:
  - "documentdb problem"
  - "documentdb issue"
  - "documentdb not working"
  - "documentdb error"
  - "docdb problem"
owner: devops-agent
objective: "Provide a systematic approach to diagnosing unclassified DocumentDB issues"
context: "This catch-all runbook covers DocumentDB issues that don't clearly fit into specific categories. It provides a general diagnostic workflow for cluster health, connectivity, performance, and configuration."
---

## Phase 1 — Triage

MUST:
- Describe the cluster: `aws docdb describe-db-clusters --db-cluster-identifier <cluster-id>`
- Check instance health: `aws docdb describe-db-instances --filters Name=db-cluster-id,Values=<cluster-id>`
- Check recent events: `aws docdb describe-events --source-type db-cluster --duration 1440`
- Verify basic connectivity and authentication

SHOULD:
- Check CloudWatch metrics for anomalies (CPU, memory, connections, latency)
- Review CloudTrail for recent API changes
- Check for pending maintenance actions: `aws docdb describe-pending-maintenance-actions`
- Verify parameter group settings

MAY:
- Check AWS Health Dashboard for service issues
- Review cluster configuration for best practices
- Run db.serverStatus() for comprehensive server diagnostics

## Phase 2 — Remediate

MUST:
- Address the identified root cause based on triage findings
- Route to the appropriate specific runbook if the issue is identified
- Verify cluster returns to healthy state after remediation

SHOULD:
- Document the issue and resolution
- Set up monitoring to detect similar issues proactively
- Review cluster configuration against best practices

MAY:
- Implement automated health checks
- Create CloudWatch dashboards for DocumentDB monitoring

## Common Issues

- symptoms: "Cluster in maintenance state"
  diagnosis: "Pending maintenance action is being applied."
  resolution: "Wait for maintenance to complete. Check describe-pending-maintenance-actions for details."

- symptoms: "Unexpected cluster behavior after engine upgrade"
  diagnosis: "Engine version change may have altered feature behavior or defaults."
  resolution: "Review engine version release notes. Check parameter group for version-specific changes."

- symptoms: "General performance degradation"
  diagnosis: "Multiple factors may contribute — instance sizing, query patterns, connection management."
  resolution: "Check CPU, memory, connections, and query profiler. Route to specific performance runbooks."

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: cluster
    content: "<cluster status and configuration>"
  - type: events
    content: "<recent cluster events>"
  - type: metrics
    content: "<relevant CloudWatch metrics>"
severity: MEDIUM
mitigation:
  immediate: "Address the identified root cause"
  long_term: "Implement comprehensive monitoring and alerting"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | MEDIUM — General troubleshooting may expose cluster configuration and metrics |
| Blast Radius | LOW — Diagnostic commands are read-only |
| Rollback Complexity | LOW — No changes during triage |
| Compliance Scope | MEDIUM — Database troubleshooting supports availability requirements |

## Escalation Conditions

- Escalate to the specific runbook owner once the problem domain is identified
- Escalate to DBA team for performance or data issues
- Escalate to AWS Support if systematic issues affect DocumentDB operations

## Data Sensitivity

- **Cluster configuration** reveals database architecture
- **CloudWatch metrics** contain operational data
- **Event logs** may contain maintenance and error details

## PROHIBITED Actions

- Do NOT modify cluster configuration during initial triage
- Do NOT run db.serverStatus() output through unprotected channels
- Do NOT share cluster endpoints or connection details outside authorized teams

## Phase 3 — Rollback

- No rollback needed for diagnostic operations
- Revert any configuration changes made during troubleshooting
- Route to the appropriate specific runbook for remediation

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
