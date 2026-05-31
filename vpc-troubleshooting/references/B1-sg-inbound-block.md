---
title: "B1 — Security Group Inbound Block"
description: "Diagnose connectivity failures caused by missing inbound security group rules"
status: active
severity: HIGH
triggers:
  - "Connection refused"
  - "Connection timed out"
  - "Can't reach instance on port"
  - "REJECT in flow logs"
owner: devops-agent
objective: "Identify missing inbound SG rules and restore access"
context: "Security groups are stateful firewalls with ALLOW rules only. Default inbound: deny all. If inbound is allowed, return traffic is automatically allowed regardless of outbound rules."
---

## Phase 1 — Triage

MUST:
- Identify the target resource's security group(s): `aws ec2 describe-instances --instance-ids <id>` → SecurityGroups
- List inbound rules: `aws ec2 describe-security-groups --group-ids <sg-id>`
- Check if the source IP/CIDR and destination port are allowed in any inbound rule
- Remember: multiple SGs on one ENI are UNION (all rules combined)

SHOULD:
- Check flow logs for REJECT on the destination ENI
- Verify the source — is it the actual client IP or a NAT/proxy IP?
- Check if the SG references another SG as source (only works within same VPC or peered VPC)

MAY:
- Check CloudTrail for recent SG rule changes
- Use Reachability Analyzer to confirm SG is the blocking point

## Common Issues

- symptoms: "Can't SSH to instance, port 22 not in SG"
  diagnosis: "Inbound rule for TCP/22 missing or restricted to wrong source CIDR."
  resolution: "Add inbound rule: TCP/22 from your IP or CIDR."

- symptoms: "SG allows 0.0.0.0/0 but still can't connect"
  diagnosis: "SG is not the issue. Check NACLs, route tables, instance state, and OS-level firewall."
  resolution: "Investigate other layers: NACL, routing, OS firewall (iptables/Windows Firewall)."

## Safety Ratings

```
safety_ratings:
  - "describe-instances: GREEN — read-only instance metadata inspection"
  - "describe-security-groups: GREEN — read-only SG rule inspection"
  - "describe-flow-logs: GREEN — read-only flow log query"
  - "authorize-security-group-ingress (add inbound rule): YELLOW — adds SG rule, recoverable by revoking"
  - "revoke-security-group-ingress (remove rule): YELLOW — removes SG rule, recoverable by re-adding"
```

## Escalation Conditions

- "Fix requires modifying production security group rules"
- "Fix requires opening inbound access from broad CIDR ranges"
- "Multiple instances or services affected by the SG change"
- "Fix involves security groups shared across multiple resources"

## Data Sensitivity

- HIGH: security group rules (expose network architecture and allowed access patterns)
- HIGH: VPC flow logs (contain source/destination IPs and ports)
- MEDIUM: instance metadata and ENI configurations

## Prohibited Actions

- "NEVER suggest 0.0.0.0/0 inbound on security groups"
- "NEVER suggest removing all NACL rules to test connectivity"
- "NEVER suggest disabling VPC flow logs"
- "NEVER suggest deleting a VPC without confirming all resources are removed"
- "NEVER add inbound rules with overly broad source CIDRs without justification"

## Phase 3 — Rollback

- If an inbound rule was added via `authorize-security-group-ingress`: revoke it with `aws ec2 revoke-security-group-ingress --group-id <sg-id> --protocol <proto> --port <port> --cidr <cidr>`
- If an inbound rule was removed via `revoke-security-group-ingress`: re-add it with `aws ec2 authorize-security-group-ingress --group-id <sg-id> --protocol <proto> --port <port> --cidr <cidr>`
- Document all existing SG rules before making changes

## Output Format

```yaml
root_cause: "sg_inbound_block — <missing rule detail>"
evidence:
  - type: security_group
    content: "<SG ID and missing rule>"
severity: HIGH
mitigation:
  immediate: "Add the required inbound rule"
  long_term: "Use IaC for SG management, implement least-privilege rules"
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
  - command: "describe-security-groups"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-network-acls"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-route-tables"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound rules as a fix"
  - "NEVER suggest disabling NACLs to troubleshoot"
  - "NEVER remove all route table entries"
