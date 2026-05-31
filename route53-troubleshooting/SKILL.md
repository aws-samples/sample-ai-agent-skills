---
name: route53-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon Route 53 problems
  by analyzing hosted zones, DNS records, health checks, routing policies,
  Resolver endpoints, domain registration, and following structured runbooks.
  Activate when: DNS resolution failures, propagation delays, NXDOMAIN errors,
  SERVFAIL responses, health check false positives, failover not triggering,
  weighted routing imbalance, geolocation routing issues, latency-based routing
  problems, private hosted zone resolution failures, split-horizon DNS issues,
  Resolver endpoint failures, DNSSEC validation errors, domain transfer failures,
  alias vs CNAME confusion, TTL caching issues, CloudFront or ELB alias record
  problems, NS delegation issues, or the user says something is wrong with
  Route 53 or DNS without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with Route 53, Route 53 Resolver, EC2 (VPC),
  CloudWatch, CloudTrail, ELB, CloudFront, and ACM permissions. Route 53 API
  endpoint is in us-east-1 (global service).
---

# Route 53 Diagnostics

## When to use

Any Route 53 investigation where the console alone is insufficient — DNS resolution debugging, health check analysis, routing policy troubleshooting, private hosted zone issues, Resolver endpoint configuration, domain registration and transfer problems, DNSSEC issues, or integration with CloudFront/ELB.

## Investigation workflow

### Step 1 — Collect and triage

```
aws route53 list-hosted-zones
aws route53 list-hosted-zones-by-name --dns-name <domain-name>
aws route53 get-hosted-zone --id <hosted-zone-id>
aws route53 list-resource-record-sets --hosted-zone-id <hosted-zone-id>
aws route53 test-dns-answer --hosted-zone-id <hosted-zone-id> --record-name <name> --record-type <type>
dig <domain-name> @8.8.8.8
dig <domain-name> @ns-xxx.awsdns-xx.com
nslookup <domain-name>
```

### Step 2 — Domain deep dive

```
aws route53 get-health-check-status --health-check-id <health-check-id>
aws route53 list-health-checks
aws route53 get-health-check --health-check-id <health-check-id>
aws route53 list-resource-record-sets --hosted-zone-id <hosted-zone-id> --query "ResourceRecordSets[?Type=='A']"
aws route53 get-hosted-zone-count
aws route53domains get-domain-detail --domain-name <domain-name>
aws route53resolver list-resolver-endpoints
aws route53resolver list-resolver-rules
```

### Step 3 — Detailed investigation

```
aws route53 get-dnssec --hosted-zone-id <hosted-zone-id>
aws route53 get-change --id <change-id>
aws route53resolver get-resolver-endpoint --resolver-endpoint-id <endpoint-id>
aws route53resolver list-resolver-endpoint-ip-addresses --resolver-endpoint-id <endpoint-id>
aws ec2 describe-vpcs --vpc-ids <vpc-id> --query 'Vpcs[*].{Id:VpcId,DnsHostnames:EnableDnsHostnames,DnsSupport:EnableDnsSupport}'
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=route53.amazonaws.com --max-results 10
aws cloudwatch get-metric-statistics --namespace AWS/Route53 --metric-name HealthCheckStatus --dimensions Name=HealthCheckId,Value=<health-check-id> --start-time <start> --end-time <end> --period 60 --statistics Minimum
aws cloudwatch get-metric-statistics --namespace AWS/Route53 --metric-name DNSQueries --dimensions Name=HostedZoneId,Value=<hosted-zone-id> --start-time <start> --end-time <end> --period 3600 --statistics Sum
```

Read `references/route53-guardrails.md` before concluding on any Route 53 issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `list-hosted-zones` | List all hosted zones in the account |
| `get-hosted-zone` | Hosted zone details including NS records and VPC associations |
| `list-resource-record-sets` | All DNS records in a hosted zone |
| `test-dns-answer` | Simulate Route 53 DNS response (authoritative answer) |
| `get-health-check-status` | Current health check status from all regions |
| `get-health-check` | Health check configuration details |
| `list-health-checks` | All health checks in the account |
| `get-dnssec` | DNSSEC signing status for a hosted zone |
| `get-change` | Status of a pending change batch (PENDING or INSYNC) |
| `route53domains get-domain-detail` | Domain registration details and transfer lock |
| `route53resolver list-resolver-endpoints` | Resolver inbound/outbound endpoints |
| `route53resolver list-resolver-rules` | Resolver forwarding rules |
| `dig` | External DNS resolution testing |
| `nslookup` | Basic DNS lookup |

## Gotchas: Route 53

- Alias vs CNAME: Alias records are free (no query charges for alias to AWS resources), work at the zone apex (example.com), and are resolved server-side by Route 53. CNAME records CANNOT be created at the zone apex (RFC violation), incur standard query charges, and add an extra DNS lookup. Always prefer alias for AWS resources.
version: "1.0.0"
last_updated: "2025-04-12"
- Health check evaluation: Standard health checks evaluate every 30 seconds. Fast health checks evaluate every 10 seconds (higher cost). Health checkers run from multiple AWS regions simultaneously. A health check is considered healthy when the configured threshold of regions report healthy (default 3 of 8+).
- Failover routing requires health checks: Failover routing policies MUST have health checks associated with the primary record. Without a health check, Route 53 always returns the primary record and never fails over. The secondary record health check is optional but recommended.
- Private hosted zones require VPC association AND enableDnsHostnames + enableDnsSupport: A private hosted zone must be associated with a VPC, AND the VPC must have both enableDnsHostnames and enableDnsSupport set to true. Missing either setting causes resolution failures silently.
- Split-horizon DNS: You can have the same domain name in both a public and private hosted zone. Instances in the associated VPC resolve the private zone; external clients resolve the public zone. This is intentional but can cause confusion during debugging.
- TTL caching means changes aren't instant: DNS resolvers cache responses for the TTL duration. Even after Route 53 propagates a change (typically 60 seconds), clients may see stale data until their cached TTL expires. Lower TTL before planned changes, then raise it after.
- DNSSEC signing vs validation: Route 53 supports DNSSEC signing for public hosted zones (you enable it). DNSSEC validation is done by resolvers (e.g., Route 53 Resolver can validate). Signing and validation are separate concerns. Enabling signing requires creating a KSK and establishing a chain of trust with the parent zone.
- Route 53 Resolver endpoints: Inbound endpoints allow on-premises DNS resolvers to forward queries TO Route 53 Resolver. Outbound endpoints allow Route 53 Resolver to forward queries TO on-premises or other DNS resolvers. Each endpoint requires at least 2 IP addresses in different AZs.
- Weighted routing zero-weight behavior: A record with weight 0 receives NO traffic UNLESS all records in the group have weight 0, in which case traffic is distributed equally among all weight-0 records. This is useful for testing — set weight to 0 to stop traffic, or set all to 0 for equal distribution.
- Geolocation vs geoproximity vs latency routing: Geolocation routes based on the geographic location of the user (continent/country/state). Geoproximity routes based on geographic distance with optional bias. Latency-based routes to the region with lowest network latency. Geolocation has a mandatory default record for unmatched locations.
- NS record delegation: When delegating a subdomain to a different hosted zone, create NS records in the parent zone pointing to the child zone's name servers. The NS records in the child zone are authoritative. Mismatched NS records between parent and child cause resolution failures.
- Domain transfer lock: Domains have a transfer lock by default. You must disable the transfer lock AND get an authorization code to transfer a domain. Some TLDs have additional requirements. Transfers can take up to 10 days. Recently registered or transferred domains have a 60-day lock.
- Route 53 is a global service: The Route 53 API endpoint is in us-east-1. Health checks, hosted zones, and domain registration are global resources. CloudWatch metrics for health checks are in us-east-1. Always use --region us-east-1 for CloudWatch health check metrics.

## Anti-hallucination rules

1. Always cite specific hosted zone configurations, record sets, health check statuses, or dig/nslookup output as evidence.
2. CNAME records CANNOT exist at the zone apex. Never suggest creating a CNAME at the zone apex — use an alias record instead.
3. DNS changes are NOT instant for end users. Even after Route 53 propagates (INSYNC), resolvers cache for the TTL duration. Never claim changes are immediately visible to all clients.
4. Failover routing REQUIRES a health check on the primary record. Never suggest failover routing without configuring a health check.
5. Private hosted zones require BOTH VPC association AND enableDnsHostnames + enableDnsSupport on the VPC. Never troubleshoot private zone resolution without checking all three.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 30 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — DNS Resolution | A1-A4 | Resolution failures, propagation delays, NXDOMAIN, SERVFAIL |
| B — Health Checks | B1-B3 | False positives, health check configuration, calculated health checks |
| C — Routing Policies | C1-C4 | Failover not triggering, weighted distribution, geolocation, latency-based |
| D — Private Hosted Zones | D1-D3 | VPC association, split-horizon, Resolver rules |
| E — Resolver | E1-E3 | Inbound endpoint issues, outbound forwarding, DNSSEC validation |
| F — Domain Management | F1-F3 | Transfer failures, registration issues, delegation |
| G — Records | G1-G3 | Alias vs CNAME, record creation errors, TTL issues |
| H — Integration | H1-H2 | CloudFront, ELB alias records |
| Z — Catch-All | Z1 | General troubleshooting |
