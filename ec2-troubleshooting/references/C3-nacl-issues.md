---
title: "C3 — Network ACL Issues"
description: "Diagnose connectivity issues caused by NACL rules"
status: active
severity: MEDIUM
triggers:
  - "NACL.*deny"
  - "network ACL.*block"
  - "stateless.*firewall"
owner: devops-agent
objective: "Identify the NACL rule blocking traffic and restore connectivity"
context: "NACLs are stateless firewalls at the subnet level. They have numbered rules evaluated in order (lowest first). They support both ALLOW and DENY rules. Default NACL allows all traffic. Custom NACLs deny all by default."
---

## Phase 1 — Triage

MUST:
- Identify the subnet: `aws ec2 describe-instances --instance-ids <id>` → SubnetId
- Get NACL for the subnet: `aws ec2 describe-network-acls --filters Name=association.subnet-id,Values=<subnet-id>`
- Check BOTH inbound AND outbound rules (NACLs are stateless)
- Verify ephemeral port range (1024-65535) is allowed for return traffic

SHOULD:
- Check NACLs on BOTH source and destination subnets
- Verify rule ordering (lower number = higher priority, first match wins)

## Guardrails

- NACLs are STATELESS. You must allow BOTH directions explicitly.
- Rule 100 ALLOW + Rule 200 DENY = traffic ALLOWED (lower number wins).
- Default NACL allows all. Custom NACLs deny all by default (rule * DENY).
- Ephemeral ports (1024-65535) must be allowed for return traffic.
- NACLs apply to ALL instances in the subnet, not per-instance.

## Common Issues

- symptoms: "Traffic blocked despite security group allowing it"
  diagnosis: "NACL is denying the traffic. NACLs are evaluated before security groups."
  resolution: "Add NACL allow rule with lower number than any deny rule matching the traffic."

- symptoms: "Outbound works but responses don't come back"
  diagnosis: "NACL outbound allows the request but inbound blocks the ephemeral port return traffic."
  resolution: "Add inbound NACL rule allowing TCP/UDP on ports 1024-65535 from the destination."

## Safety Ratings
Rate each remediation action in Phase 2:
- GREEN: read-only diagnostic commands (describe-*, get-*, list-*)
- YELLOW: state-changing but recoverable (modify security group, restart instance, modify parameter)
- RED: destructive or irreversible (terminate instance, delete volume, modify key infrastructure)

Format as:
```
safety_ratings:
  - "describe-network-acls, describe-subnets: GREEN — read-only"
  - "Add NACL allow rule: YELLOW — affects all instances in subnet, recoverable by deleting rule"
  - "Reorder NACL rules (add lower-numbered rule): YELLOW — changes evaluation order, recoverable"
  - "Delete NACL deny rule: YELLOW — broadens access for entire subnet, recoverable by re-adding"
  - "Replace subnet NACL association: YELLOW — changes all rules at once for subnet, recoverable"
```

## Escalation Conditions
- NACL is shared across multiple subnets with different security requirements
- Fix requires allowing broad CIDR ranges that may violate compliance policies
- NACL changes affect production subnets with sensitive workloads
- Rule limit (20 inbound + 20 outbound default) is reached
- NACL is managed by IaC and manual changes would cause drift

## Data Sensitivity
- HIGH: describe-network-acls (reveals all subnet-level firewall rules and deny lists)
- MEDIUM: describe-subnets (reveals subnet CIDRs and AZ placement)
- MEDIUM: VPC flow logs (reveals traffic patterns and rejected connections)

## Prohibited Actions
- NEVER suggest replacing a custom NACL with the default NACL (allows all) as a permanent fix
- NEVER suggest adding allow-all rules (0.0.0.0/0 all ports) to troubleshoot
- NEVER suggest deleting deny rules without understanding why they were added
- NEVER suggest modifying NACLs on subnets you don't own or understand the purpose of

## Phase 3 — Rollback
- If NACL rule was added: delete the rule with `aws ec2 delete-network-acl-entry --network-acl-id <id> --rule-number <num> --ingress|--egress`
- If NACL rule was deleted: re-create with `aws ec2 create-network-acl-entry` using original parameters
- If NACL association was changed: re-associate original NACL with `aws ec2 replace-network-acl-association`
- If rule ordering was changed: delete new rule and restore original rule numbers

## Output Format

```yaml
root_cause: "NACL — <inbound_deny|outbound_deny|ephemeral_blocked|rule_ordering>"
evidence:
  - type: nacl_rules
    content: "<NACL rules showing the block>"
severity: MEDIUM
mitigation:
  immediate: "Add or reorder NACL rules"
  long_term: "Use security groups as primary firewall, NACLs only for subnet-level deny"
```

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-instances"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-console-output"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "ssm send-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound security group rules as a fix"
  - "NEVER suggest disabling instance metadata service"
  - "NEVER terminate instances without confirmation"
