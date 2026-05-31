---
title: "Z1 — General Neptune Troubleshooting"
description: "Catch-all runbook for Neptune issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers:
  - "neptune problem"
  - "neptune issue"
  - "neptune not working"
  - "neptune error"
owner: devops-agent
objective: "Provide a systematic approach to diagnosing unclassified Neptune issues"
context: "This catch-all runbook covers Neptune issues that don't clearly fit into specific categories."
---

## Phase 1 — Triage

MUST:
- Describe the cluster: `aws neptune describe-db-clusters --db-cluster-identifier <cluster-id>`
- Check cluster events: `aws neptune describe-events --source-type db-cluster --duration 1440`
- Check CloudWatch metrics for anomalies (CPU, memory, connections, latency)
- Verify cluster status: `curl https://<cluster-endpoint>:8182/status`

SHOULD:
- Check for pending maintenance: `aws neptune describe-pending-maintenance-actions`
- Review CloudTrail for recent changes: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=rds.amazonaws.com --max-results 20`
- Check AWS Health Dashboard for service issues
- Verify engine version: `aws neptune describe-db-clusters --db-cluster-identifier <cluster-id> --query 'DBClusters[0].EngineVersion'`

MAY:
- Run diagnostic queries via Gremlin/SPARQL
- Check for region-specific issues
- Review cluster configuration against best practices

## Phase 2 — Remediate

MUST:
- Address the identified root cause
- Route to the appropriate specific runbook if identified
- Verify cluster returns to healthy state

SHOULD:
- Document the issue and resolution
- Set up monitoring for similar issues
- Review configuration against best practices

MAY:
- Implement automated health checks
- Create CloudWatch dashboards for Neptune

## Common Issues

- symptoms: "Cluster in modifying state"
  diagnosis: "A modification operation is in progress."
  resolution: "Wait for the operation to complete. Check cluster events for details."

- symptoms: "Engine version upgrade available"
  diagnosis: "A newer engine version is available with bug fixes or features."
  resolution: "Plan engine upgrade during maintenance window. Test in non-production first."

- symptoms: "General performance degradation"
  diagnosis: "Multiple factors may contribute."
  resolution: "Check CPU, memory, buffer cache hit ratio, and query patterns. Route to specific runbooks."

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: cluster
    content: "<cluster status and configuration>"
  - type: events
    content: "<recent events>"
  - type: metrics
    content: "<relevant CloudWatch metrics>"
severity: MEDIUM
mitigation:
  immediate: "Address the identified root cause"
  long_term: "Implement comprehensive monitoring"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: MEDIUM — general troubleshooting may involve inspecting cluster configurations
- Service Disruption Risk: LOW — diagnostic commands are read-only
- Compliance Impact: LOW — general troubleshooting follows standard operational procedures

## Escalation Conditions
- Issue cannot be resolved using standard troubleshooting procedures
- Cluster enters an unrecoverable state
- Multiple unrelated issues occur simultaneously indicating platform problem
- AWS Health Dashboard shows Neptune service event
- Issue requires AWS Support engagement

## Data Sensitivity
- Cluster configuration — INTERNAL
- CloudWatch metrics and logs — INTERNAL
- Audit logs — CONFIDENTIAL
- Error messages and stack traces — INTERNAL
- Never expose database contents during troubleshooting

## Prohibited Actions
- Do NOT make changes to production clusters without a change management process
- Do NOT run destructive commands during initial diagnosis
- Do NOT share cluster details in public forums without sanitizing
- Do NOT bypass established escalation procedures
- Do NOT attempt fixes outside the scope of Neptune service boundaries
- Do NOT ignore recurring issues — document patterns for root cause analysis

## Phase 3 — Rollback
- Revert any diagnostic changes made during troubleshooting
- Restore cluster to last known good configuration if changes were applied
- Document all troubleshooting steps taken and their outcomes
- Verify cluster health after troubleshooting session
- Update runbooks with new findings
- Close out any temporary access granted for troubleshooting

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
