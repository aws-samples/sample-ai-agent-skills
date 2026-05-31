---
name: route53-resolver-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon Route 53
  Resolver problems by analyzing resolver endpoints, forwarding rules,
  DNS Firewall, query logging, inbound/outbound endpoints, RAM sharing,
  VPC associations, and DNSSEC validation using structured runbooks.
  Activate when: resolver endpoint failures, DNS forwarding issues,
  DNS Firewall rule problems, query logging errors, endpoint connectivity
  issues, RAM sharing failures, VPC association problems, DNSSEC
  validation errors, or the user says something is wrong with Route 53
  Resolver.
compatibility: >
  Requires AWS CLI or SDK access with route53resolver, ec2, ram,
  logs, cloudtrail, and cloudwatch permissions.
---

# Amazon Route 53 Resolver Diagnostics

## When to use

Any Route 53 Resolver investigation — resolver endpoint creation, endpoint connectivity, forwarding rules, DNS Firewall, query logging, inbound/outbound endpoints, RAM sharing, VPC associations, or DNSSEC validation.

## Investigation workflow

### Step 1 — Collect and triage

```
aws route53resolver list-resolver-endpoints --query 'ResolverEndpoints[*].{Id:Id,Name:Name,Direction:Direction,Status:Status,IpCount:IpAddressCount}'
aws route53resolver list-resolver-rules --query 'ResolverRules[*].{Id:Id,Name:Name,DomainName:DomainName,Status:Status,RuleType:RuleType}'
aws route53resolver list-firewall-rule-groups --query 'FirewallRuleGroups[*].{Id:Id,Name:Name,ShareStatus:ShareStatus}'
```

### Step 2 — Domain deep dive

```
aws route53resolver get-resolver-endpoint --resolver-endpoint-id <endpoint-id>
aws route53resolver list-resolver-endpoint-ip-addresses --resolver-endpoint-id <endpoint-id>
aws route53resolver get-firewall-rule-group --firewall-rule-group-id <group-id>
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=route53resolver.amazonaws.com --max-results 20
aws route53resolver list-resolver-query-log-configs --query 'ResolverQueryLogConfigs[*].{Id:Id,Name:Name,Status:Status,AssociationCount:AssociationCount}'
aws route53resolver list-resolver-dnssec-configs --query 'ResolverDnssecConfigs[*].{Id:Id,OwnerId:OwnerId,ResourceId:ResourceId,ValidationStatus:ValidationStatus}'
```

Read `references/guardrails.md` before concluding on any Route 53 Resolver issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `route53resolver list-resolver-endpoints` | List resolver endpoints |
| `route53resolver get-resolver-endpoint` | Get endpoint details |
| `route53resolver list-resolver-rules` | List forwarding rules |
| `route53resolver list-firewall-rule-groups` | List DNS Firewall groups |
| `route53resolver list-resolver-query-log-configs` | List query log configs |
| `route53resolver list-resolver-dnssec-configs` | List DNSSEC configs |
| `route53resolver list-resolver-endpoint-ip-addresses` | List endpoint IPs |

## Gotchas: Route 53 Resolver

- Resolver endpoints require at least 2 IP addresses in different AZs for high availability. Single-AZ endpoints are a single point of failure.
- Inbound endpoints receive DNS queries FROM on-premises networks. Outbound endpoints send DNS queries TO on-premises or external DNS servers. The naming is from the VPC's perspective.
- DNS Firewall rules are evaluated in priority order (lowest number first). A BLOCK rule at priority 100 overrides an ALLOW rule at priority 200.
- Forwarding rules are associated with VPCs, not endpoints. A rule can be associated with multiple VPCs. System rules (for AWS internal domains) take precedence over custom rules.
- Query logging captures ALL DNS queries in the VPC, not just those going through resolver endpoints. This includes queries to the VPC DNS resolver (AmazonProvidedDNS).
- DNSSEC validation is per-VPC. Enabling it causes queries for domains with invalid DNSSEC signatures to return SERVFAIL. This can break resolution for misconfigured domains.
- RAM sharing of rules and firewall groups is cross-account only within the same AWS Organization by default. Sharing outside the organization requires enabling external sharing.

## Anti-hallucination rules

1. Always cite specific endpoint IDs, rule IDs, or API responses as evidence.
2. Inbound endpoints receive queries; outbound endpoints send queries. Never reverse the direction.
3. DNS Firewall and forwarding rules are different features. Never conflate them.
4. DNSSEC validation can break DNS resolution for misconfigured domains. Never enable without testing.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 14 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Endpoints | A1–A2 | Endpoint creation, endpoint connectivity |
| B — Forwarding | B1–B2 | Forwarding rule errors, conditional forwarding |
| C — DNS Firewall | C1–C2 | Firewall rule groups, firewall domain lists |
| D — Query Logging | D1–D2 | Logging configuration, log analysis |
| E — Direction | E1–E2 | Inbound endpoint issues, outbound endpoint issues |
| F — Sharing | F1–F2 | RAM sharing errors, VPC association issues |
| G — Security | G1 | DNSSEC validation |
| Z — Catch-All | Z1 | General troubleshooting |
