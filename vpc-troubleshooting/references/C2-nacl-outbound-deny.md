---
title: "C2 — NACL Outbound Deny"
description: "Diagnose connectivity failures caused by NACL outbound deny rules"
status: active
severity: HIGH
triggers:
  - "Can't initiate connections from subnet"
  - "Return traffic blocked"
  - "NACL stateless issue"
owner: devops-agent
objective: "Identify NACL outbound rules blocking traffic"
context: "NACLs are STATELESS. Even if inbound is allowed, outbound return traffic must be explicitly allowed. This is the most common NACL misconfiguration."
---

## Phase 1 — Triage

MUST:
- Get the subnet's NACL and list OUTBOUND rules
- Check if return traffic ports are allowed (ephemeral ports 1024-65535 for most OS)
- Check if initiated outbound connections are allowed (destination port + CIDR)
- Remember: NACLs are stateless — both directions must be explicitly allowed

SHOULD:
- Check flow logs for outbound REJECT entries
- Verify ephemeral port range for the OS (Linux: 32768-60999, Windows: 49152-65535)
- Check if the NACL allows outbound to the VPC DNS (port 53)

MAY:
- Compare with working subnets' NACL outbound rules
- Check if a recent security hardening removed outbound rules

## Common Issues

- symptoms: "Inbound SSH works but responses don't reach client"
  diagnosis: "NACL outbound doesn't allow ephemeral ports (1024-65535) for return traffic."
  resolution: "Add outbound allow rule for TCP 1024-65535 to 0.0.0.0/0."

- symptoms: "Instance can't make API calls, outbound HTTPS blocked"
  diagnosis: "NACL outbound doesn't allow TCP 443."
  resolution: "Add outbound allow rule for TCP 443 to 0.0.0.0/0 (or specific CIDR)."

## Safety Ratings

```
safety_ratings:
  - "describe-network-acls: GREEN — read-only NACL rule inspection"
  - "describe-flow-logs: GREEN — read-only flow log query"
  - "create-network-acl-entry (add outbound allow): YELLOW — adds NACL rule, recoverable by deleting the entry"
  - "replace-network-acl-entry (modify outbound rule): YELLOW — changes existing rule, recoverable by replacing back"
  - "delete-network-acl-entry (remove outbound deny): YELLOW — removes NACL rule, recoverable by re-adding"
```

## Escalation Conditions

- "Fix requires changing NACL rules (stateless, affects all traffic)"
- "Multiple subnets or AZs affected"
- "Fix involves ephemeral port rules — affects return traffic for all connections"
- "NACL changes affect subnets with production workloads"

## Data Sensitivity

- HIGH: NACL rules (expose network architecture and subnet-level access controls)
- HIGH: VPC flow logs (contain source/destination IPs and ports)
- MEDIUM: subnet CIDR ranges and outbound traffic patterns

## Prohibited Actions

- "NEVER suggest 0.0.0.0/0 inbound on security groups"
- "NEVER suggest removing all NACL rules to test connectivity"
- "NEVER suggest disabling VPC flow logs"
- "NEVER suggest deleting a VPC without confirming all resources are removed"
- "NEVER modify NACL to deny all outbound (can break connectivity for entire subnet)"

## Phase 3 — Rollback

- If an outbound NACL rule was added via `create-network-acl-entry`: delete it with `aws ec2 delete-network-acl-entry --network-acl-id <nacl-id> --rule-number <num> --egress`
- If an outbound NACL rule was replaced: replace it back with `aws ec2 replace-network-acl-entry --network-acl-id <nacl-id> --rule-number <num> --protocol <proto> --port-range From=<port>,To=<port> --cidr-block <cidr> --rule-action <allow|deny> --egress`
- If an outbound NACL rule was deleted: re-add it with `aws ec2 create-network-acl-entry` with the original parameters and `--egress`
- Document all outbound NACL rules before making changes

## Output Format

```yaml
root_cause: "nacl_outbound_deny — <rule detail>"
evidence:
  - type: network_acl
    content: "<NACL ID, missing outbound rule>"
severity: HIGH
mitigation:
  immediate: "Add outbound allow rules for required traffic and ephemeral return ports"
  long_term: "Always configure NACL rules in pairs (inbound + outbound)"
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
