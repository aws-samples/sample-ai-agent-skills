---
title: "E1 — Inbound Endpoint Issues"
description: "Diagnose inbound resolver endpoint issues for receiving DNS queries from external networks"
status: active
severity: HIGH
triggers: ["inbound endpoint not receiving", "on-premises cannot resolve", "inbound DNS failed"]
owner: devops-agent
objective: "Resolve inbound endpoint issues preventing external DNS query reception"
context: "Inbound endpoints receive DNS queries FROM on-premises or external networks. On-premises DNS servers forward queries to inbound endpoint IP addresses. Requires security groups allowing inbound DNS (TCP/UDP 53)."
---

## Phase 1 — Triage

MUST:
- Check inbound endpoint status: `aws route53resolver get-resolver-endpoint --resolver-endpoint-id <endpoint-id>`
- List endpoint IP addresses: `aws route53resolver list-resolver-endpoint-ip-addresses --resolver-endpoint-id <endpoint-id>`
- Verify security group allows inbound DNS: `aws ec2 describe-security-groups --group-ids <sg-id>`

SHOULD:
- Verify on-premises DNS servers are configured to forward to the endpoint IPs
- Check VPN or Direct Connect connectivity between on-premises and VPC

MAY:
- Test DNS resolution from on-premises: `dig @<endpoint-ip> <aws-domain>`
- Check VPC flow logs for DNS traffic to endpoint ENIs

## Phase 2 — Remediate

MUST:
- Add inbound security group rules for TCP/UDP 53 from on-premises CIDR
- Verify route table has return path to on-premises network
- Ensure endpoint IPs are reachable from on-premises (VPN/DX routing)

SHOULD:
- Configure on-premises DNS servers to forward specific AWS domains to endpoint IPs
- Verify NACLs allow DNS traffic and ephemeral port responses

MAY:
- Add additional IP addresses for capacity: `aws route53resolver associate-resolver-endpoint-ip-address --resolver-endpoint-id <endpoint-id> --ip-address SubnetId=<subnet-id>`
- Set up monitoring for inbound query volume

## Common Issues

- symptoms: "On-premises DNS queries to inbound endpoint time out"
  diagnosis: "Security group missing inbound DNS rules or VPN/DX routing issue."
  resolution: "Add inbound TCP/UDP 53 rules from on-premises CIDR. Verify VPN/DX route propagation."

- symptoms: "Inbound endpoint resolves some domains but not others"
  diagnosis: "Private hosted zones not associated with the endpoint VPC."
  resolution: "Associate private hosted zones with the VPC containing the inbound endpoint."

## Safety Ratings

```
safety_ratings:
  - "get-resolver-endpoint: GREEN — read-only endpoint details"
  - "list-resolver-endpoint-ip-addresses: GREEN — read-only IP listing"
  - "authorize-security-group-ingress: YELLOW — modifies network access"
  - "associate-resolver-endpoint-ip-address: YELLOW — adds ENI to endpoint"
```

## Escalation Conditions

- Inbound endpoint unreachable despite correct security groups and routing
- On-premises network team cannot configure DNS forwarding
- VPN or Direct Connect connectivity issues affecting DNS traffic
- Endpoint capacity insufficient for query volume

## Data Sensitivity

- Endpoint IP addresses: MEDIUM — used by on-premises DNS for forwarding
- Security group rules: MEDIUM — reveal network access policy
- VPC flow logs: HIGH — contain source/destination IPs and ports
- DNS query results: MEDIUM — reveal internal AWS resource names

## Prohibited Actions

- NEVER expose inbound endpoint IPs to the public internet
- NEVER remove security group rules without verifying no other services depend on them
- NEVER modify VPN or Direct Connect routing without network team coordination
- NEVER delete inbound endpoints that on-premises DNS servers are actively using

## Phase 3 — Rollback

1. If security group rules were added incorrectly, revoke them
2. If endpoint IP addresses were added, disassociate if not needed: `aws route53resolver disassociate-resolver-endpoint-ip-address --resolver-endpoint-id <endpoint-id> --ip-address SubnetId=<subnet-id>,Ip=<ip>`
3. If route table changes were made, revert to previous configuration
4. Verify on-premises DNS resolution is restored
5. Document all network changes

## Output Format

```yaml
root_cause: "inbound_endpoint — <specific_cause>"
evidence:
  - type: inbound_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix inbound endpoint connectivity"
  long_term: "Monitor inbound endpoint health and capacity"
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
  - "NEVER suggest disabling DNSSEC without understanding impact"
  - "NEVER suggest wildcard DNS records without review"
  - "NEVER modify hosted zones without backup"
