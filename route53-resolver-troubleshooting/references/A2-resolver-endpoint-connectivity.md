---
title: "A2 — Resolver Endpoint Connectivity"
description: "Diagnose resolver endpoint connectivity and DNS resolution failures"
status: active
severity: HIGH
triggers: ["endpoint not resolving", "DNS timeout", "resolver connectivity failed"]
owner: devops-agent
objective: "Resolve resolver endpoint connectivity issues"
context: "Resolver endpoints use ENIs with security groups. Inbound endpoints need inbound DNS (53) access. Outbound endpoints need outbound DNS (53) access to target servers. NACLs and route tables also affect connectivity."
---

## Phase 1 — Triage

MUST:
- Check endpoint status: `aws route53resolver get-resolver-endpoint --resolver-endpoint-id <endpoint-id>`
- List endpoint IP addresses: `aws route53resolver list-resolver-endpoint-ip-addresses --resolver-endpoint-id <endpoint-id>`
- Check security group rules: `aws ec2 describe-security-groups --group-ids <sg-id> --query 'SecurityGroups[*].{Inbound:IpPermissions,Outbound:IpPermissionsEgress}'`

SHOULD:
- Verify NACL rules allow DNS traffic (TCP/UDP 53) in both directions
- Check route table for path to target DNS servers (outbound) or source networks (inbound)

MAY:
- Test DNS resolution from an EC2 instance in the same VPC: `dig @<endpoint-ip> <domain>`
- Check VPC flow logs for dropped DNS packets

## Phase 2 — Remediate

MUST:
- For inbound endpoints: allow inbound TCP/UDP 53 from on-premises CIDR ranges in security group
- For outbound endpoints: allow outbound TCP/UDP 53 to target DNS server IPs in security group
- Verify route table has routes to the target network (VPN, Direct Connect, or peering)

SHOULD:
- Ensure NACLs allow ephemeral ports (1024-65535) for DNS response traffic
- Verify DNS target servers are reachable and responding

MAY:
- Enable VPC flow logs to diagnose packet-level connectivity issues
- Add CloudWatch alarms for resolver endpoint query metrics

## Common Issues

- symptoms: "DNS queries to inbound endpoint time out from on-premises"
  diagnosis: "Security group missing inbound rule for DNS (TCP/UDP 53) from on-premises CIDR."
  resolution: "Add inbound rules for TCP/UDP 53 from on-premises network CIDR ranges."

- symptoms: "Outbound forwarding rule queries fail with SERVFAIL"
  diagnosis: "Outbound endpoint cannot reach target DNS servers — routing or security group issue."
  resolution: "Verify route table has path to target DNS servers. Check security group outbound rules."

## Safety Ratings

```
safety_ratings:
  - "get-resolver-endpoint: GREEN — read-only endpoint details"
  - "describe-security-groups: GREEN — read-only security group rules"
  - "authorize-security-group-ingress: YELLOW — modifies network access rules"
  - "create-flow-logs: YELLOW — creates logging resources with cost implications"
```

## Escalation Conditions

- Endpoint connectivity fails despite correct security groups and route tables
- DNS resolution works intermittently suggesting capacity or network issues
- On-premises network team cannot configure DNS forwarding to endpoint IPs
- VPC flow logs show packets reaching endpoint but no response

## Data Sensitivity

- `aws route53resolver list-resolver-endpoint-ip-addresses`: MEDIUM — reveals endpoint IPs used for DNS
- `aws ec2 describe-security-groups`: MEDIUM — reveals network security configuration
- VPC flow logs: HIGH — contain source/destination IPs and ports for all traffic
- DNS query results: MEDIUM — may reveal internal domain names and IP addresses

## Prohibited Actions

- NEVER open security groups to 0.0.0.0/0 for DNS to troubleshoot connectivity
- NEVER disable NACLs entirely to test DNS connectivity
- NEVER modify route tables without understanding the full network topology
- NEVER share endpoint IP addresses publicly or in unsecured documentation

## Phase 3 — Rollback

1. If security group rules were added incorrectly, revoke them: `aws ec2 revoke-security-group-ingress --group-id <sg-id> --protocol udp --port 53 --cidr <cidr>`
2. If route table entries were added, remove them if they cause routing conflicts
3. If flow logs were enabled, disable them if no longer needed to avoid costs
4. Verify DNS resolution is restored after changes
5. Document all network changes for the infrastructure team

## Output Format

```yaml
root_cause: "endpoint_connectivity — <specific_cause>"
evidence:
  - type: connectivity_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix endpoint connectivity"
  long_term: "Implement monitoring for resolver endpoint health"
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
