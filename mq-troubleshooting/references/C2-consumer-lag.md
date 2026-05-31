---
title: "C2 — Consumer Lag"
description: "Diagnose increasing consumer lag and message backlog"
status: active
severity: MEDIUM
triggers:
  - "consumer lag"
  - "message backlog"
  - "queue depth increasing"
  - "consumers behind"
  - "messages accumulating"
owner: devops-agent
objective: "Identify and resolve consumer lag issues"
context: "Consumer lag occurs when: consumers are slower than producers, consumers are disconnected, prefetch settings suboptimal, message processing errors, or insufficient consumer instances."
---

## Phase 1 — Triage

MUST:
- Check queue depth metrics: `aws cloudwatch get-metric-statistics --namespace AWS/AmazonMQ --metric-name QueueSize --dimensions Name=Broker,Value=<broker-name> Name=Queue,Value=<queue-name> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Check consumer count via web console
- Verify consumers are connected and active
- Check EnqueueCount vs DequeueCount rate

SHOULD:
- Check for consumer errors in application logs
- Verify prefetch size is appropriate
- Check for message redelivery (poison messages)
- Review consumer acknowledgment mode

MAY:
- Check if DLQ is accumulating failed messages
- Review consumer processing time per message

## Phase 2 — Remediate

MUST:
- Restart disconnected consumers
- Add more consumer instances if processing is the bottleneck
- Fix consumer errors causing message redelivery

SHOULD:
- Tune prefetch size (higher for throughput, lower for fairness)
- Implement DLQ for poison messages
- Use competing consumers pattern for parallel processing

MAY:
- Implement consumer auto-scaling
- Set message TTL to prevent indefinite accumulation

## Output Format

```yaml
root_cause: "consumer_lag — <specific_cause>"
evidence:
  - type: queue_depth
    content: "<queue size metrics>"
  - type: consumer_count
    content: "<active consumers>"
severity: MEDIUM
mitigation:
  immediate: "Restart consumers or add more instances"
  long_term: "Implement consumer monitoring and auto-scaling"
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
