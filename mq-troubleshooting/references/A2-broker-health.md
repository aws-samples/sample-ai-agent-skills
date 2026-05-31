---
title: "A2 — Broker Health Issues"
description: "Diagnose broker health degradation and availability problems"
status: active
severity: HIGH
triggers:
  - "broker unhealthy"
  - "broker not responding"
  - "broker degraded"
  - "broker restarting"
  - "failover occurred"
owner: devops-agent
objective: "Identify and resolve broker health issues"
context: "Broker health issues include: high CPU, memory pressure, network issues, storage problems, or broker in maintenance/reboot state."
---

## Phase 1 — Triage

MUST:
- Check broker state: `aws mq describe-broker --broker-id <broker-id> --query 'BrokerState'`
- Check CPU: `aws cloudwatch get-metric-statistics --namespace AWS/AmazonMQ --metric-name CpuUtilization --dimensions Name=Broker,Value=<broker-name> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Check memory/heap: `aws cloudwatch get-metric-statistics --namespace AWS/AmazonMQ --metric-name HeapUsage --dimensions Name=Broker,Value=<broker-name> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Check storage: `aws cloudwatch get-metric-statistics --namespace AWS/AmazonMQ --metric-name StorePercentUsage --dimensions Name=Broker,Value=<broker-name> --start-time <start> --end-time <end> --period 300 --statistics Maximum`

SHOULD:
- Check broker logs: `aws logs filter-log-events --log-group-name /aws/amazonmq/broker/<broker-id>/general`
- Check network metrics (NetworkIn, NetworkOut)
- Verify no pending maintenance
- Check for recent failover events (active/standby)

MAY:
- Check AWS Health Dashboard for MQ service issues
- Review broker configuration for suboptimal settings

## Phase 2 — Remediate

MUST:
- If CPU high: consider larger instance type or reduce load
- If memory high: tune heap settings or reduce message backlog
- If storage full: purge old messages or increase storage
- If broker stuck: reboot broker: `aws mq reboot-broker --broker-id <broker-id>`

SHOULD:
- Set up CloudWatch alarms for CPU > 80%, HeapUsage > 80%, StorePercentUsage > 80%
- Review and optimize message patterns

MAY:
- Scale to larger instance type (requires new broker for deployment mode change)
- Implement message TTL to prevent storage buildup

## Output Format

```yaml
root_cause: "broker_health_issue — <specific_cause>"
evidence:
  - type: broker_state
    content: "<broker state>"
  - type: metrics
    content: "<CPU, memory, storage metrics>"
severity: HIGH
mitigation:
  immediate: "Address the specific health issue"
  long_term: "Implement comprehensive monitoring with alarms"
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
