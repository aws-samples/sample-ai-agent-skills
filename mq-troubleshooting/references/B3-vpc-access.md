---
title: "B3 — VPC Access Issues"
description: "Diagnose VPC networking issues affecting broker access"
status: active
severity: MEDIUM
triggers:
  - "VPC access denied"
  - "cannot reach broker"
  - "cross-VPC access"
  - "subnet routing"
  - "private access"
owner: devops-agent
objective: "Identify and resolve VPC access issues"
context: "VPC access issues include: security group rules, subnet routing, NACLs, VPC peering, Transit Gateway, or public vs private accessibility configuration."
---

## Phase 1 — Triage

MUST:
- Check broker VPC configuration: `aws mq describe-broker --broker-id <broker-id> --query '{SubnetIds:SubnetIds,SecurityGroups:SecurityGroups,PubliclyAccessible:PubliclyAccessible}'`
- Check security group rules: `aws ec2 describe-security-groups --group-ids <sg-ids>`
- Verify client and broker are in same VPC or have network path
- Check subnet route tables: `aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=<subnet-id>`

SHOULD:
- Check NACLs for port blocking
- Verify VPC peering or Transit Gateway if cross-VPC
- Check DNS resolution of broker endpoint

MAY:
- Check VPC Flow Logs for blocked traffic
- Verify NAT Gateway if broker needs internet access

## Phase 2 — Remediate

MUST:
- Fix security group inbound rules for broker ports
- Ensure proper routing between client and broker subnets
- Fix NACL rules if blocking traffic

SHOULD:
- Set up VPC peering for cross-VPC access
- Configure DNS resolution for peered VPCs

MAY:
- Use AWS PrivateLink for secure cross-account access
- Implement VPC Flow Logs for network debugging

## Output Format

```yaml
root_cause: "vpc_access_issue — <specific_cause>"
evidence:
  - type: vpc_config
    content: "<VPC, subnets, security groups>"
  - type: routing
    content: "<route table entries>"
severity: MEDIUM
mitigation:
  immediate: "Fix security groups or routing"
  long_term: "Implement proper network architecture for broker access"
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
