---
title: "A1 — Broker Creation Failures"
description: "Diagnose why Amazon MQ broker creation is failing"
status: active
severity: HIGH
triggers:
  - "broker creation failed"
  - "CREATION_FAILED"
  - "cannot create broker"
  - "broker not starting"
  - "creation error"
owner: devops-agent
objective: "Identify why broker creation is failing and resolve the issue"
context: "Broker creation failures occur due to: IAM permissions, VPC/subnet issues, security group misconfiguration, invalid instance type, service quota exceeded, or KMS key issues."
---

## Phase 1 — Triage

MUST:
- Check broker state: `aws mq describe-broker --broker-id <broker-id> --query '{State:BrokerState,Engine:EngineType}'`
- Check CloudTrail for creation errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateBroker --max-results 10`
- Verify IAM permissions for mq:CreateBroker
- Check service quotas: `aws service-quotas get-service-quota --service-code mq --quota-code L-A0C1BEBC`

SHOULD:
- Verify subnet exists and has available IPs: `aws ec2 describe-subnets --subnet-ids <subnet-ids>`
- Check security group exists: `aws ec2 describe-security-groups --group-ids <sg-ids>`
- Verify instance type is valid for the engine
- Check KMS key permissions if using encryption

MAY:
- Verify VPC has DNS support enabled
- Check for conflicting broker names

## Phase 2 — Remediate

MUST:
- Fix IAM permissions to include mq:CreateBroker and related actions
- Ensure subnets have available IP addresses
- Use valid instance type for the engine (mq.m5.large, mq.t3.micro, etc.)

SHOULD:
- Create broker: `aws mq create-broker --broker-name <name> --engine-type <ActiveMQ|RABBITMQ> --engine-version <version> --host-instance-type <type> --deployment-mode <mode> --users '[{"username":"admin","password":"<pass>","consoleAccess":true}]' --subnet-ids <subnets> --security-groups <sgs>`

MAY:
- Request quota increase if at broker limit
- Use CloudFormation/Terraform for repeatable creation

## Output Format

```yaml
root_cause: "broker_creation_failure — <specific_cause>"
evidence:
  - type: broker_state
    content: "<broker state>"
  - type: creation_error
    content: "<CloudTrail error>"
severity: HIGH
mitigation:
  immediate: "Fix the specific creation blocker"
  long_term: "Use IaC for consistent broker provisioning"
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
