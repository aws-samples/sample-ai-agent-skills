---
title: "Z1 — General MemoryDB Troubleshooting"
description: "Catch-all runbook for MemoryDB issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers:
  - "memorydb problem"
  - "memorydb issue"
  - "memorydb not working"
  - "memorydb error"
owner: devops-agent
objective: "Provide a systematic approach to diagnosing unclassified MemoryDB issues"
context: "This catch-all runbook covers MemoryDB issues that don't clearly fit into specific categories."
---

## Phase 1 — Triage

MUST:
- Describe the cluster: `aws memorydb describe-clusters --cluster-name <cluster-name> --show-shard-details`
- Check cluster events: `aws memorydb describe-events --source-type cluster --duration 1440`
- Check CloudWatch metrics for anomalies (CPU, memory, connections, latency)
- Verify basic connectivity: `redis-cli --tls -h <endpoint> PING`

SHOULD:
- Check for pending service updates: `aws memorydb describe-service-updates`
- Review CloudTrail for recent changes
- Check AWS Health Dashboard for service issues
- Verify ACL and parameter group configuration

MAY:
- Run redis-cli INFO for comprehensive diagnostics
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
- Create CloudWatch dashboards

## Common Issues

- symptoms: "Cluster in modifying state"
  diagnosis: "A modification operation is in progress."
  resolution: "Wait for the operation to complete. Check cluster events for details."

- symptoms: "Service update pending"
  diagnosis: "AWS has released a service update that should be applied."
  resolution: "Apply the service update during a maintenance window."

- symptoms: "General performance degradation"
  diagnosis: "Multiple factors may contribute."
  resolution: "Check CPU, memory, connections, and slow log. Route to specific performance runbooks."

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

| Aspect | Rating | Notes |
|--------|--------|-------|
| Data Risk | Medium | Operations may affect service configurations and data access patterns |
| Service Impact | High | Incorrect changes can disrupt active workloads and dependent services |
| Recovery Complexity | Medium | Most actions are reversible but may require coordinated rollback procedures |
| Blast Radius | Medium | Impact typically scoped to the affected resource and its direct dependents |

## Escalation Conditions

- **Escalate immediately** if the issue involves potential data loss or corruption
- **Escalate immediately** if a production workload is impacted and cannot be restored within the agreed SLA
- **Escalate immediately** if the root cause is unclear after initial diagnostic steps are exhausted
- **Escalate to service team** if the issue appears to be a platform-level bug or undocumented behavior
- **Escalate to security team** if unauthorized access, policy violations, or credential exposure is suspected

## Data Sensitivity

- Verify data classification before performing any read or export operations
- Ensure all troubleshooting outputs (logs, configurations, snapshots) are handled according to the customer's data handling policy
- Mask or redact personally identifiable information (PII) in all shared diagnostics
- Do not copy production data to non-production environments without explicit authorization
- Retain troubleshooting artifacts only for the duration required and dispose of them securely

## Prohibited Actions

- **NEVER** delete a MemoryDB cluster without verifying snapshot backups and data durability requirements
- **NEVER** modify ACL rules on production clusters without testing authentication changes in a staging environment
- **NEVER** force a failover on a production cluster during peak traffic without customer approval
- **NEVER** reduce the number of shards or replicas on a cluster serving production workloads without capacity analysis
- **NEVER** disable encryption in-transit or at-rest on clusters handling sensitive data

## Phase 3 � Rollback

1. **Identify rollback trigger**: Determine if the applied change has caused degradation, unexpected behavior, or failed validation checks
2. **Preserve current state**: Capture logs, configuration snapshots, and resource states before initiating rollback
3. **Execute rollback steps**: Revert configuration changes in reverse order of application, verifying each step
4. **Validate service recovery**: Confirm that the service has returned to its pre-change operational state using health checks and functional tests
5. **Document rollback**: Record the rollback trigger, steps taken, time to recovery, and any residual issues for post-incident review
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
  - "NEVER suggest disabling encryption in transit"
  - "NEVER suggest removing ACL authentication"
  - "NEVER suggest public subnet placement"
