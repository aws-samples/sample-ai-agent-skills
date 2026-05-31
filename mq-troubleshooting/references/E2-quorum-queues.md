---
title: "E2 — Quorum Queue Issues"
description: "Diagnose RabbitMQ quorum queue problems"
status: active
severity: MEDIUM
triggers:
  - "quorum queue error"
  - "queue replication"
  - "Raft consensus"
  - "quorum queue slow"
  - "queue not available"
owner: devops-agent
objective: "Identify and resolve quorum queue issues"
context: "Quorum queue issues include: cluster deployment required, higher latency than classic queues, feature limitations (no per-message TTL), leader election issues, or insufficient cluster nodes."
---

## Phase 1 — Triage

MUST:
- Verify broker is RabbitMQ cluster: `aws mq describe-broker --broker-id <broker-id> --query '{Engine:EngineType,DeploymentMode:DeploymentMode}'`
- Check queue type via Management UI (must be x-queue-type: quorum)
- Verify cluster has 3 nodes for quorum
- Check queue leader distribution across nodes

SHOULD:
- Check queue replication status
- Verify consumer and publisher performance
- Check for leader election issues in logs

MAY:
- Compare quorum queue vs classic queue performance
- Check Raft log size

## Phase 2 — Remediate

MUST:
- Ensure cluster deployment mode (quorum queues require 3 nodes)
- Declare queues with x-queue-type: quorum argument
- Monitor queue leader distribution

SHOULD:
- Balance queue leaders across nodes
- Set appropriate delivery-limit for poison message handling
- Use quorum queues for critical data, classic for ephemeral

MAY:
- Tune quorum queue settings for performance
- Implement queue policies for consistent configuration

## Output Format

```yaml
root_cause: "quorum_queue_issue — <specific_cause>"
evidence:
  - type: deployment_mode
    content: "<cluster configuration>"
  - type: queue_status
    content: "<quorum queue status>"
severity: MEDIUM
mitigation:
  immediate: "Fix queue configuration or cluster issues"
  long_term: "Implement quorum queue best practices"
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

- **NEVER** force-reboot a production broker during active message processing without draining connections
- **NEVER** delete a broker with persistent messages that have not been consumed or backed up
- **NEVER** modify authentication credentials on production brokers without coordinating with all connected clients
- **NEVER** change network-of-brokers topology during peak message throughput periods
- **NEVER** downgrade broker engine versions or instance types on production brokers without capacity testing

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
