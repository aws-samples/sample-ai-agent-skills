---
title: "B2 — Security Group Outbound Block"
description: "Diagnose connectivity failures caused by missing outbound security group rules"
status: active
severity: HIGH
triggers:
  - "Can't reach external service"
  - "DNS resolution fails"
  - "API calls timing out from instance"
owner: devops-agent
objective: "Identify missing outbound SG rules and restore connectivity"
context: "Default SG outbound: allow all. Custom SGs may restrict outbound. If outbound is allowed, return traffic is automatically allowed (stateful). Removing the default outbound allow-all rule blocks all initiated connections."
---

## Phase 1 — Triage

MUST:
- Get the resource's security group(s) and list outbound rules
- Check if the destination IP/CIDR and port are allowed in outbound rules
- Verify the default outbound rule (0.0.0.0/0 allow all) hasn't been removed
- Check if DNS (UDP/TCP 53) is allowed outbound (required for name resolution)

SHOULD:
- Check flow logs for REJECT on the source ENI (outbound direction)
- Verify if the SG was recently modified
- Check if outbound is restricted to specific endpoints (common in secure environments)

MAY:
- Check if a compliance tool or automation removed the default outbound rule
- Verify if VPC endpoints could bypass the outbound restriction

## Common Issues

- symptoms: "Instance can't resolve DNS, outbound restricted"
  diagnosis: "Outbound rule doesn't allow UDP/TCP 53 to VPC DNS (CIDR+2 address)."
  resolution: "Add outbound rule for UDP/TCP 53 to the VPC CIDR or 0.0.0.0/0."

- symptoms: "Application can't reach external API after SG hardening"
  diagnosis: "Default outbound allow-all was removed and specific API endpoint not added."
  resolution: "Add outbound rule for the specific destination IP/CIDR and port."

## Safety Ratings

```
safety_ratings:
  - "describe-security-groups: GREEN — read-only SG rule inspection"
  - "describe-flow-logs: GREEN — read-only flow log query"
  - "authorize-security-group-egress (add outbound rule): YELLOW — adds SG rule, recoverable by revoking"
  - "revoke-security-group-egress (remove rule): YELLOW — removes SG rule, recoverable by re-adding"
```

## Escalation Conditions

- "Fix requires modifying production security group rules"
- "Fix requires restoring default outbound allow-all rule"
- "Multiple instances or services affected by the SG change"
- "Fix involves security groups managed by compliance automation"

## Data Sensitivity

- HIGH: security group rules (expose network architecture and allowed access patterns)
- HIGH: VPC flow logs (contain source/destination IPs and ports)
- MEDIUM: instance metadata and outbound destination details

## Prohibited Actions

- "NEVER suggest 0.0.0.0/0 inbound on security groups"
- "NEVER suggest removing all NACL rules to test connectivity"
- "NEVER suggest disabling VPC flow logs"
- "NEVER suggest deleting a VPC without confirming all resources are removed"
- "NEVER restore default outbound allow-all without confirming compliance requirements"

## Phase 3 — Rollback

- If an outbound rule was added via `authorize-security-group-egress`: revoke it with `aws ec2 revoke-security-group-egress --group-id <sg-id> --protocol <proto> --port <port> --cidr <cidr>`
- If an outbound rule was removed via `revoke-security-group-egress`: re-add it with `aws ec2 authorize-security-group-egress --group-id <sg-id> --protocol <proto> --port <port> --cidr <cidr>`
- Document all existing SG outbound rules before making changes

## Output Format

```yaml
root_cause: "sg_outbound_block — <missing rule detail>"
evidence:
  - type: security_group
    content: "<SG ID and missing outbound rule>"
severity: HIGH
mitigation:
  immediate: "Add the required outbound rule"
  long_term: "Document required outbound destinations before restricting default rule"
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
