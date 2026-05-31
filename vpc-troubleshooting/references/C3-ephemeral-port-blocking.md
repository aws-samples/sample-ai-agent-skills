---
title: "C3 — Ephemeral Port Blocking"
description: "Diagnose connectivity failures caused by blocked ephemeral/return ports in NACLs"
status: active
severity: HIGH
triggers:
  - "Connection established but no data returned"
  - "Intermittent connectivity"
  - "NACL ephemeral ports"
owner: devops-agent
objective: "Fix ephemeral port blocking in NACLs"
context: "When a client connects to a server, the response goes back to a random ephemeral port on the client. NACLs are stateless, so the return traffic on ephemeral ports must be explicitly allowed. Different OS use different ranges."
---

## Phase 1 — Triage

MUST:
- Identify the NACL on the CLIENT subnet (where return traffic arrives)
- Check inbound rules for ephemeral port range (1024-65535 covers all OS)
- Identify the NACL on the SERVER subnet (where return traffic leaves)
- Check outbound rules for ephemeral port range

SHOULD:
- Determine the OS ephemeral range: Linux 32768-60999, Windows 49152-65535, NAT gateway 1024-65535
- Check flow logs for REJECT on ephemeral ports
- Verify both directions: client→server AND server→client

MAY:
- Test with a broad ephemeral range (1024-65535) to confirm the issue, then narrow down

## Common Issues

- symptoms: "Can connect to RDS but queries time out"
  diagnosis: "NACL allows outbound 3306 but doesn't allow inbound ephemeral ports for the response."
  resolution: "Add inbound allow for TCP 1024-65535 from the RDS subnet CIDR."

- symptoms: "NAT gateway traffic intermittently fails"
  diagnosis: "NACL on NAT subnet doesn't allow inbound on ports 1024-65535 from internet."
  resolution: "NAT gateway uses ports 1024-65535. Allow inbound TCP/UDP 1024-65535 from 0.0.0.0/0."

## Safety Ratings

```
safety_ratings:
  - "describe-network-acls: GREEN — read-only NACL rule inspection"
  - "describe-flow-logs: GREEN — read-only flow log query"
  - "create-network-acl-entry (add ephemeral port allow): YELLOW — adds NACL rule, recoverable by deleting"
  - "replace-network-acl-entry (widen port range): YELLOW — modifies existing rule, recoverable by replacing back"
```

## Escalation Conditions

- "Fix requires changing NACL rules (stateless, affects all traffic)"
- "Multiple subnets or AZs affected by ephemeral port blocking"
- "Fix involves widening port ranges — increases attack surface"
- "NAT gateway traffic affected — impacts all private subnet internet access"

## Data Sensitivity

- HIGH: NACL rules (expose network architecture and subnet-level access controls)
- HIGH: VPC flow logs (contain source/destination IPs and ports)
- MEDIUM: ephemeral port ranges and OS-specific configurations

## Prohibited Actions

- "NEVER suggest 0.0.0.0/0 inbound on security groups"
- "NEVER suggest removing all NACL rules to test connectivity"
- "NEVER suggest disabling VPC flow logs"
- "NEVER suggest deleting a VPC without confirming all resources are removed"
- "NEVER open ephemeral ports wider than 1024-65535"

## Phase 3 — Rollback

- If an ephemeral port NACL rule was added: delete it with `aws ec2 delete-network-acl-entry --network-acl-id <nacl-id> --rule-number <num> --ingress` (or `--egress`)
- If a NACL rule was replaced to widen port range: replace it back with `aws ec2 replace-network-acl-entry` using the original port range
- Document all NACL rules on both client and server subnets before making changes

## Output Format

```yaml
root_cause: "ephemeral_port_blocking — <NACL and port range>"
evidence:
  - type: network_acl
    content: "<NACL ID and missing ephemeral port rule>"
severity: HIGH
mitigation:
  immediate: "Add allow rules for ephemeral ports 1024-65535"
  long_term: "Standard practice: always allow 1024-65535 in NACLs for return traffic"
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
