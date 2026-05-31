---
title: "B1 — Connection Failures"
description: "Diagnose client connection failures to Amazon MQ brokers"
status: active
severity: HIGH
triggers:
  - "connection refused"
  - "connection timeout"
  - "cannot connect to broker"
  - "client connection error"
  - "broker unreachable"
owner: devops-agent
objective: "Identify why clients cannot connect to the broker"
context: "Connection failures occur due to: wrong endpoint, security group blocking ports, VPC routing issues, authentication failures, TLS mismatch, or broker not in RUNNING state."
---

## Phase 1 — Triage

MUST:
- Check broker state and endpoints: `aws mq describe-broker --broker-id <broker-id> --query '{State:BrokerState,Endpoints:BrokerInstances[*].Endpoints}'`
- Verify security group allows client port: `aws ec2 describe-security-groups --group-ids <sg-id>`
- Check broker engine for correct port (ActiveMQ: 61617, RabbitMQ: 5671)
- Verify client is in same VPC or has network path to broker

SHOULD:
- Test network connectivity to broker endpoint and port
- Verify client credentials are correct
- Check if broker is publicly accessible: `aws mq describe-broker --broker-id <broker-id> --query 'PubliclyAccessible'`
- Check for VPC peering or Transit Gateway if cross-VPC

MAY:
- Check NACLs for port blocking
- Review client connection string format
- Check for connection limit exhaustion

## Phase 2 — Remediate

MUST:
- Add security group rule for broker port from client CIDR
- Use correct endpoint URL from describe-broker output
- Ensure broker is in RUNNING state
- Use correct credentials

SHOULD:
- Use failover transport for ActiveMQ: `failover:(ssl://broker-1:61617,ssl://broker-2:61617)`
- Use amqps:// scheme for RabbitMQ TLS connections

MAY:
- Enable public accessibility for development (not recommended for production)
- Set up VPC peering for cross-VPC access

## Output Format

```yaml
root_cause: "connection_failure — <specific_cause>"
evidence:
  - type: broker_endpoints
    content: "<broker endpoints>"
  - type: security_groups
    content: "<security group rules>"
severity: HIGH
mitigation:
  immediate: "Fix security groups, endpoint, or credentials"
  long_term: "Implement connection monitoring and failover"
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
