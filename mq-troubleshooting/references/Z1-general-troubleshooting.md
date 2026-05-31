---
title: "Z1 — General Amazon MQ Troubleshooting"
description: "Catch-all systematic investigation for Amazon MQ issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers:
  - "Amazon MQ issue"
  - "broker problem"
  - "MQ not working"
  - "MQ error"
  - "general troubleshooting"
owner: devops-agent
objective: "Systematically investigate Amazon MQ issues using a structured diagnostic approach"
context: "This runbook provides a general-purpose investigation framework for Amazon MQ issues that don't clearly match a specific runbook."
---

## Phase 1 — Triage

MUST:
- Check broker overview: `aws mq describe-broker --broker-id <broker-id>`
- Check broker state, engine type, deployment mode
- Check key metrics: CPU, HeapUsage, StorePercentUsage, CurrentConnectionsCount
- Check broker logs: `aws logs filter-log-events --log-group-name /aws/amazonmq/broker/<broker-id>/general`

SHOULD:
- Check recent CloudTrail events for broker changes
- Review configuration for recent modifications
- Check for pending maintenance or upgrades
- Verify network connectivity to broker

MAY:
- Check AWS Health Dashboard for MQ service issues
- Review billing for unexpected usage

## Phase 2 — Remediate

MUST:
- Based on triage findings, pivot to the appropriate specific runbook:
  - Broker issues → A1 (creation), A2 (health), A3 (storage)
  - Connectivity → B1 (connection), B2 (TLS), B3 (VPC)
  - Performance → C1 (throughput), C2 (consumer lag), C3 (memory)
  - ActiveMQ → D1 (specific), D2 (network of brokers)
  - RabbitMQ → E1 (specific), E2 (quorum queues)
  - Maintenance → F1 (upgrades), F2 (configuration)
  - Security → G1 (authentication), G2 (encryption)
- Document findings and root cause

SHOULD:
- Check for multiple concurrent issues
- Review recent changes

MAY:
- Set up comprehensive monitoring
- Create post-incident report

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: broker_status
    content: "<broker state and configuration>"
  - type: metrics
    content: "<key metrics>"
  - type: logs
    content: "<relevant log entries>"
severity: MEDIUM
general_analysis:
  broker_state: "RUNNING | degraded | failing"
  engine: "ActiveMQ | RabbitMQ"
  specific_runbook: "<recommended runbook ID>"
mitigation:
  immediate: "Follow the identified specific runbook"
  long_term: "Implement comprehensive monitoring and alerting"
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
