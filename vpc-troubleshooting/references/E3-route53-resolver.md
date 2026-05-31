---
title: "E3 — Route 53 Resolver Issues"
description: "Diagnose DNS forwarding failures with Route 53 Resolver endpoints and rules"
status: active
severity: HIGH
triggers:
  - "DNS forwarding not working"
  - "On-premises DNS resolution fails"
  - "Resolver endpoint errors"
owner: devops-agent
objective: "Fix Route 53 Resolver forwarding and hybrid DNS resolution"
context: "Route 53 Resolver enables hybrid DNS: inbound endpoints allow on-premises to resolve VPC DNS, outbound endpoints allow VPC to resolve on-premises DNS. Rules define which domains are forwarded."
---

## Phase 1 — Triage

MUST:
- Check resolver endpoints: `aws route53resolver list-resolver-endpoints`
- Verify endpoint status is OPERATIONAL
- Check resolver rules: `aws route53resolver list-resolver-rules`
- Verify rules are associated with the correct VPCs

SHOULD:
- Check security groups on resolver endpoints (inbound: allow DNS from on-prem; outbound: allow DNS to on-prem DNS servers)
- Verify on-premises DNS servers are reachable from the resolver endpoint subnets
- Check if VPN/Direct Connect is up (required for hybrid DNS)

MAY:
- Test DNS resolution targeting the resolver endpoint IP directly
- Check CloudWatch metrics for resolver query counts and failures

## Common Issues

- symptoms: "Outbound resolver can't reach on-premises DNS"
  diagnosis: "Security group on outbound endpoint doesn't allow UDP/TCP 53 to on-prem DNS IPs."
  resolution: "Add outbound rule for UDP/TCP 53 to on-premises DNS server IPs."

- symptoms: "Resolver rule not taking effect"
  diagnosis: "Rule not associated with the VPC where resolution is attempted."
  resolution: "Associate the resolver rule with the correct VPC."

## Safety Ratings

```
safety_ratings:
  - "list-resolver-endpoints: GREEN — read-only resolver endpoint listing"
  - "list-resolver-rules: GREEN — read-only resolver rule listing"
  - "get-resolver-endpoint: GREEN — read-only endpoint detail check"
  - "associate-resolver-rule (associate rule with VPC): YELLOW — adds rule association, recoverable by disassociating"
  - "authorize-security-group-ingress (allow DNS on endpoint SG): YELLOW — adds SG rule, recoverable by revoking"
  - "disassociate-resolver-rule: YELLOW — removes rule association, recoverable by re-associating"
```

## Escalation Conditions

- "Fix requires modifying Route 53 Resolver endpoints in production"
- "Fix requires changing security groups on resolver endpoints"
- "Hybrid DNS resolution affected — impacts on-premises to VPC communication"
- "Multiple VPCs affected by resolver rule association issues"

## Data Sensitivity

- HIGH: resolver endpoint IPs and configurations (expose hybrid DNS architecture)
- HIGH: resolver rules (expose DNS forwarding targets and on-premises DNS servers)
- MEDIUM: security group rules on resolver endpoints, VPC associations

## Prohibited Actions

- "NEVER suggest 0.0.0.0/0 inbound on security groups"
- "NEVER suggest removing all NACL rules to test connectivity"
- "NEVER suggest disabling VPC flow logs"
- "NEVER suggest deleting a VPC without confirming all resources are removed"
- "NEVER delete resolver endpoints without confirming hybrid DNS is no longer needed"

## Phase 3 — Rollback

- If a resolver rule was associated with a VPC: disassociate it with `aws route53resolver disassociate-resolver-rule --resolver-rule-id <rule-id> --vpc-id <vpc-id>`
- If a security group rule was added to resolver endpoint: revoke it with `aws ec2 revoke-security-group-ingress --group-id <sg-id> --protocol <proto> --port <port> --cidr <cidr>`
- If a resolver rule was disassociated: re-associate it with `aws route53resolver associate-resolver-rule --resolver-rule-id <rule-id> --vpc-id <vpc-id>`
- Document resolver endpoint IPs, rule associations, and security group rules before changes

## Output Format

```yaml
root_cause: "route53_resolver — <detail>"
evidence:
  - type: resolver
    content: "<endpoint status and rule configuration>"
severity: HIGH
mitigation:
  immediate: "Fix endpoint security groups or rule associations"
  long_term: "Monitor resolver endpoints, ensure redundant endpoints across AZs"
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
