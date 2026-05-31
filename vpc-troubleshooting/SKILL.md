---
name: vpc-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS VPC networking
  problems by analyzing network configurations, flow logs, and
  following structured runbooks. Activate when: resources can't
  communicate within a VPC, internet access is broken, cross-VPC
  connectivity fails (peering, Transit Gateway), DNS resolution
  fails, VPN or Direct Connect is down, security groups or NACLs
  are blocking traffic, NAT gateway issues, VPC endpoints not
  working, subnet IP exhaustion, route table misconfiguration,
  flow logs show unexpected REJECT entries, or the user says
  something is wrong with VPC networking without naming specific
  symptoms.
compatibility: >
  Requires AWS CLI or SDK access with VPC, EC2, Route53, CloudWatch,
  and optionally Transit Gateway, VPN, Direct Connect permissions.
---

# VPC Diagnostics

## When to use

Any VPC networking investigation where basic connectivity checks are insufficient — route table analysis, security group/NACL rule evaluation, flow log analysis, DNS resolution, NAT/IGW configuration, peering/TGW routing, VPN tunnel state, endpoint policies, or subnet capacity.

## Investigation workflow

### Step 1 — Collect and triage

```
# Identify the VPC and affected resources
aws ec2 describe-vpcs --vpc-ids <vpc-id>
aws ec2 describe-subnets --filters Name=vpc-id,Values=<vpc-id>

# Check route tables
aws ec2 describe-route-tables --filters Name=vpc-id,Values=<vpc-id>

# Check security groups
aws ec2 describe-security-groups --filters Name=vpc-id,Values=<vpc-id>

# Check NACLs
aws ec2 describe-network-acls --filters Name=vpc-id,Values=<vpc-id>

# Use Reachability Analyzer for end-to-end path validation
aws ec2 create-network-insights-path --source <src-eni> --destination <dst> --protocol TCP --destination-port <port>
aws ec2 start-network-insights-analysis --network-insights-path-id <path-id>
```

### Step 2 — Domain deep dive

```
# Flow logs
aws logs filter-log-events --log-group-name <flow-log-group> --filter-pattern "REJECT"

# DNS
aws ec2 describe-vpc-attribute --vpc-id <vpc-id> --attribute enableDnsSupport
aws route53resolver list-resolver-rules

# NAT / IGW
aws ec2 describe-nat-gateways --filter Name=vpc-id,Values=<vpc-id>
aws ec2 describe-internet-gateways --filters Name=attachment.vpc-id,Values=<vpc-id>

# Peering / TGW
aws ec2 describe-vpc-peering-connections --filters Name=requester-vpc-info.vpc-id,Values=<vpc-id>
aws ec2 describe-transit-gateway-attachments --filters Name=resource-id,Values=<vpc-id>

# VPC Endpoints
aws ec2 describe-vpc-endpoints --filters Name=vpc-id,Values=<vpc-id>
```

Read `references/vpc-networking-guardrails.md` before concluding on any VPC issue.

### Step 3 — Detailed path

```
# VPN tunnels
aws ec2 describe-vpn-connections --vpn-connection-ids <id>

# Direct Connect
aws directconnect describe-connections
aws directconnect describe-virtual-interfaces

# Transit Gateway route tables
aws ec2 search-transit-gateway-routes --transit-gateway-route-table-id <id> --filters Name=type,Values=static,propagated
```

## Gotchas: VPC Networking

- Security groups are STATEFUL (return traffic auto-allowed). NACLs are STATELESS (both directions required).
- Security groups have ALLOW rules only. Use NACLs for DENY rules.
- NACL rules are evaluated in order (lowest number first). First match wins.
- Default NACL allows all. Custom NACLs deny all by default.
- VPC peering is NON-TRANSITIVE. A↔B and B↔C does NOT mean A↔C. Use Transit Gateway.
- VPC peering does NOT support overlapping CIDRs.
- NAT gateway must be in a PUBLIC subnet (with IGW route). Private subnet instances route 0.0.0.0/0 → NAT.
- VPC endpoints (Gateway type: S3, DynamoDB) are free. Interface endpoints cost per hour + per GB.
- Gateway endpoints use route table entries. Interface endpoints use DNS and ENIs.
- DNS resolution requires enableDnsSupport=true. Private hosted zones require enableDnsHostnames=true.
- Subnets are public if their route table has 0.0.0.0/0 → IGW. There is no "public subnet" attribute.
- Each subnet has 5 reserved IPs (first 4 + last 1). A /24 subnet has 251 usable IPs, not 256.
- VPC flow logs show ACCEPT/REJECT at ENI level. They do NOT show packet content or which rule matched.
- MTU: default 1500. Jumbo frames (9001) only within same VPC. Clamped to 1500 through IGW/NAT/VPN/peering.
- Transit Gateway has its own route tables separate from VPC route tables. Both must be configured.

### Traffic evaluation order

```
Route table → NACL (stateless) → Security group (stateful)
```

### Subnet classification

| Route for 0.0.0.0/0 | Subnet type |
|----------------------|-------------|
| → IGW | Public |
| → NAT gateway | Private (with internet) |
| No route | Isolated (no internet) |

## Anti-hallucination rules

1. Always cite specific route table entries, SG/NACL rules, or flow log entries as evidence.
2. Security groups have NO deny rules. Never suggest adding a deny rule to a SG.
3. NACLs are stateless. Always check BOTH inbound AND outbound rules.
4. VPC peering is non-transitive. Never claim traffic can hop through a peered VPC.
5. A subnet is public because of its route table, not because of any attribute.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 38 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Routing | A1-A4 | Missing routes, blackhole routes, longest prefix match, asymmetric routing |
| B — Security Groups | B1-B3 | Inbound block, outbound block, cross-VPC SG references |
| C — NACLs | C1-C3 | Inbound deny, outbound deny, ephemeral port blocking |
| D — NAT / IGW | D1-D4 | NAT gateway failures, IGW detached, NAT port exhaustion, NAT bandwidth |
| E — DNS | E1-E4 | VPC DNS disabled, private hosted zone, Route 53 Resolver, split-horizon |
| F — Peering / TGW | F1-F4 | Peering routes, TGW routes, overlapping CIDRs, cross-region peering |
| G — VPN / DX | G1-G3 | VPN tunnel down, BGP issues, Direct Connect VLAN |
| H — VPC Endpoints | H1-H3 | Gateway endpoint routing, interface endpoint DNS, endpoint policy |
| I — Subnet / IP | I1-I3 | IP exhaustion, subnet sizing, secondary CIDR |
| J — Flow Logs | J1-J2 | Flow log analysis, REJECT investigation |
| K — MTU / Performance | K1-K2 | MTU fragmentation, cross-AZ latency |
| Z — Catch-All | Z1 | General troubleshooting |
