---
title: "J2 — Flow Log REJECT Investigation"
description: "Systematically investigate REJECT entries in VPC flow logs"
status: active
severity: HIGH
triggers:
  - "REJECT in flow logs"
  - "Traffic being blocked"
  - "Unexpected REJECT entries"
owner: devops-agent
objective: "Identify the source of REJECT entries and fix the blocking rule"
context: "REJECT in flow logs means traffic was blocked by either a security group or NACL. Flow logs don't tell you WHICH one blocked it. You must check both. Evaluation order: Route table → NACL → Security group."
---

## Phase 1 — Triage

MUST:
- Extract the REJECT entry details: source IP, dest IP, source port, dest port, protocol
- Identify the ENI and its associated instance/resource
- Check security group rules for the ENI (inbound for incoming, outbound for outgoing)
- Check NACL rules for the subnet (both inbound AND outbound — stateless)

SHOULD:
- Determine if the REJECT is for initiated traffic or return traffic
- Check if ephemeral ports are blocked in NACL (common for return traffic)
- Verify the traffic should actually be allowed (might be legitimate blocking)

MAY:
- Check if the REJECT is from a port scan or unauthorized access attempt
- Correlate with CloudTrail for SG/NACL changes around the time of first REJECT

## Common Issues

- symptoms: "REJECT on ephemeral port range, SG allows the service port"
  diagnosis: "NACL blocking return traffic on ephemeral ports (stateless)."
  resolution: "Add NACL rule to allow ephemeral ports 1024-65535."

- symptoms: "REJECT on port 443 outbound, SG has no outbound rules"
  diagnosis: "Default outbound allow-all was removed from the security group."
  resolution: "Add outbound rule for TCP 443 or restore the default allow-all outbound."

## Safety Ratings

```
safety_ratings:
  - "describe-flow-logs: GREEN — read-only flow log query"
  - "describe-security-groups: GREEN — read-only SG rule inspection"
  - "describe-network-acls: GREEN — read-only NACL rule inspection"
  - "authorize-security-group-ingress (add missing allow rule): YELLOW — adds SG rule, recoverable by revoking"
  - "create-network-acl-entry (add missing NACL allow): YELLOW — adds NACL rule, recoverable by deleting"
```

## Escalation Conditions

- "REJECT entries indicate production traffic being blocked"
- "Fix requires modifying production security group or NACL rules"
- "REJECT pattern suggests unauthorized access attempts — security review needed"
- "Multiple resources affected by the same blocking rule"

## Data Sensitivity

- HIGH: VPC flow logs (contain source/destination IPs, ports, and REJECT reasons)
- HIGH: security group rules (expose allowed access patterns)
- HIGH: NACL rules (expose subnet-level access controls)
- MEDIUM: ENI details, resource associations

## Prohibited Actions

- "NEVER suggest 0.0.0.0/0 inbound on security groups"
- "NEVER suggest removing all NACL rules to test connectivity"
- "NEVER suggest disabling VPC flow logs"
- "NEVER suggest deleting a VPC without confirming all resources are removed"
- "NEVER allow traffic that was intentionally blocked by security policy"

## Phase 3 — Rollback

- If an SG inbound rule was added: revoke it with `aws ec2 revoke-security-group-ingress --group-id <sg-id> --protocol <proto> --port <port> --cidr <cidr>`
- If an SG outbound rule was added: revoke it with `aws ec2 revoke-security-group-egress --group-id <sg-id> --protocol <proto> --port <port> --cidr <cidr>`
- If a NACL rule was added: delete it with `aws ec2 delete-network-acl-entry --network-acl-id <nacl-id> --rule-number <num> --ingress` (or `--egress`)
- Document all SG rules and NACL rules before making changes

## Output Format

```yaml
root_cause: "flow_log_reject — <SG or NACL rule detail>"
evidence:
  - type: flow_log
    content: "<REJECT entry and matching rule analysis>"
severity: HIGH
mitigation:
  immediate: "Add the missing allow rule in SG or NACL"
  long_term: "Monitor flow logs for REJECT patterns, automate alerting"
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
