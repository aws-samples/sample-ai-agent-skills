---
title: "E2 — Outbound Endpoint Issues"
description: "Diagnose outbound resolver endpoint issues for forwarding DNS queries to external servers"
status: active
severity: HIGH
triggers: ["outbound endpoint not forwarding", "DNS forwarding failed", "outbound DNS timeout"]
owner: devops-agent
objective: "Resolve outbound endpoint issues preventing DNS query forwarding"
context: "Outbound endpoints forward DNS queries FROM the VPC TO on-premises or external DNS servers. They are used by forwarding rules. Requires security groups allowing outbound DNS (TCP/UDP 53) to target servers."
---

## Phase 1 — Triage

MUST:
- Check outbound endpoint status: `aws route53resolver get-resolver-endpoint --resolver-endpoint-id <endpoint-id>`
- List endpoint IP addresses: `aws route53resolver list-resolver-endpoint-ip-addresses --resolver-endpoint-id <endpoint-id>`
- Verify security group allows outbound DNS: `aws ec2 describe-security-groups --group-ids <sg-id>`

SHOULD:
- Check forwarding rules using this endpoint: `aws route53resolver list-resolver-rules --query 'ResolverRules[?ResolverEndpointId==\`<endpoint-id>\`]'`
- Verify target DNS servers are reachable from the VPC

MAY:
- Test DNS resolution to target servers from an instance in the same subnet
- Check VPC flow logs for outbound DNS traffic from endpoint ENIs

## Phase 2 — Remediate

MUST:
- Add outbound security group rules for TCP/UDP 53 to target DNS server IPs
- Verify route table has path to target DNS servers (VPN, DX, or internet)
- Ensure target DNS servers are running and accepting queries

SHOULD:
- Verify NACLs allow outbound DNS and inbound response traffic
- Check that target DNS servers are not rate-limiting queries from the endpoint

MAY:
- Add additional IP addresses for capacity scaling
- Configure health monitoring for target DNS servers

## Common Issues

- symptoms: "Forwarding rules return SERVFAIL for all forwarded domains"
  diagnosis: "Outbound endpoint cannot reach target DNS servers."
  resolution: "Check security group outbound rules. Verify route table has path to target DNS server network."

- symptoms: "Forwarding works intermittently with timeouts"
  diagnosis: "Target DNS servers overloaded or network path unstable."
  resolution: "Add multiple target IPs in forwarding rules. Check VPN/DX stability."

## Safety Ratings

```
safety_ratings:
  - "get-resolver-endpoint: GREEN — read-only endpoint details"
  - "describe-security-groups: GREEN — read-only security group rules"
  - "authorize-security-group-egress: YELLOW — modifies outbound network access"
  - "associate-resolver-endpoint-ip-address: YELLOW — adds ENI to endpoint"
```

## Escalation Conditions

- Outbound endpoint cannot reach target DNS servers despite correct routing
- Target DNS servers are managed by external team and are unresponsive
- Forwarding failures cause application DNS resolution outages
- Endpoint capacity insufficient for forwarded query volume

## Data Sensitivity

- Endpoint IP addresses: LOW — source IPs for outbound DNS queries
- Target DNS server IPs: MEDIUM — reveals on-premises DNS infrastructure
- Security group rules: MEDIUM — reveal network access policy
- Forwarding rule domains: MEDIUM — reveal DNS architecture

## Prohibited Actions

- NEVER open all outbound ports to troubleshoot DNS forwarding
- NEVER point forwarding rules to untrusted DNS servers
- NEVER delete outbound endpoints that active forwarding rules depend on
- NEVER modify routing without understanding impact on other VPC traffic

## Phase 3 — Rollback

1. If security group rules were modified, revert to previous configuration
2. If endpoint IP addresses were changed, restore original configuration
3. If forwarding rule target IPs were updated, revert to original targets
4. Verify DNS forwarding is restored for all affected domains
5. Document all changes for the network team

## Output Format

```yaml
root_cause: "outbound_endpoint — <specific_cause>"
evidence:
  - type: outbound_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix outbound endpoint connectivity"
  long_term: "Implement target DNS server health monitoring"
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
