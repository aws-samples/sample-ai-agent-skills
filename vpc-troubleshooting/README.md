# VPC Networking Diagnostics Skill

Agent skill for investigating and troubleshooting AWS VPC networking problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for VPC networking when basic connectivity checks aren't enough — route table analysis, security group/NACL evaluation, flow log analysis, DNS resolution, NAT/IGW configuration, peering/TGW routing, VPN tunnel state, endpoint policies, and subnet capacity.

### Activate When

- Resources can't communicate within a VPC
- Internet access is broken (NAT, IGW)
- Cross-VPC connectivity fails (peering, Transit Gateway)
- DNS resolution fails (VPC DNS, private hosted zones, Route 53 Resolver)
- VPN or Direct Connect is down
- Security groups or NACLs are blocking traffic
- VPC endpoints not working (gateway or interface)
- Subnet IP exhaustion
- Route table misconfiguration
- Flow logs show unexpected REJECT entries
- MTU/fragmentation issues
- The user says something is wrong with VPC networking

---

## Skill Structure

```
vpc-troubleshooting/
├── SKILL.md                          # Main skill definition and investigation workflow
├── README.md                         # This file
└── references/
    ├── A1-missing-routes.md
    ├── A2-blackhole-routes.md
    ├── A3-longest-prefix-match.md
    ├── A4-asymmetric-routing.md
    ├── B1-sg-inbound-block.md
    ├── B2-sg-outbound-block.md
    ├── B3-cross-vpc-sg-reference.md
    ├── C1-nacl-inbound-deny.md
    ├── C2-nacl-outbound-deny.md
    ├── C3-ephemeral-port-blocking.md
    ├── D1-nat-gateway-failures.md
    ├── D2-igw-detached.md
    ├── D3-nat-port-exhaustion.md
    ├── D4-nat-bandwidth.md
    ├── E1-vpc-dns-disabled.md
    ├── E2-private-hosted-zone.md
    ├── E3-route53-resolver.md
    ├── E4-split-horizon-dns.md
    ├── F1-peering-routes.md
    ├── F2-tgw-routes.md
    ├── F3-overlapping-cidrs.md
    ├── F4-cross-region-peering.md
    ├── G1-vpn-tunnel-down.md
    ├── G2-bgp-issues.md
    ├── G3-direct-connect-vlan.md
    ├── H1-gateway-endpoint-routing.md
    ├── H2-interface-endpoint-dns.md
    ├── H3-endpoint-policy.md
    ├── I1-ip-exhaustion.md
    ├── I2-subnet-sizing.md
    ├── I3-secondary-cidr.md
    ├── J1-flow-log-analysis.md
    ├── J2-reject-investigation.md
    ├── K1-mtu-fragmentation.md
    ├── K2-cross-az-latency.md
    ├── Z1-general-troubleshooting.md
    ├── vpc-networking-guardrails.md
    └── vpc-hallucination-patterns.yaml
```

---

## Runbook Library (38 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Routing** | A1–A4 | Missing routes, blackhole routes, longest prefix match, asymmetric routing |
| **B — Security Groups** | B1–B3 | Inbound block, outbound block, cross-VPC SG references |
| **C — NACLs** | C1–C3 | Inbound deny, outbound deny, ephemeral port blocking |
| **D — NAT / IGW** | D1–D4 | NAT gateway failures, IGW detached, NAT port exhaustion, NAT bandwidth |
| **E — DNS** | E1–E4 | VPC DNS disabled, private hosted zone, Route 53 Resolver, split-horizon |
| **F — Peering / TGW** | F1–F4 | Peering routes, TGW routes, overlapping CIDRs, cross-region peering |
| **G — VPN / DX** | G1–G3 | VPN tunnel down, BGP issues, Direct Connect VLAN |
| **H — VPC Endpoints** | H1–H3 | Gateway endpoint routing, interface endpoint DNS, endpoint policy |
| **I — Subnet / IP** | I1–I3 | IP exhaustion, subnet sizing, secondary CIDR |
| **J — Flow Logs** | J1–J2 | Flow log analysis, REJECT investigation |
| **K — MTU / Performance** | K1–K2 | MTU fragmentation, cross-AZ latency |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails & Anti-Hallucination

### Networking Guardrails (`vpc-networking-guardrails.md`)
12 rules covering: SG stateful vs NACL stateless, SG allow-only rules, NACL rule ordering, public subnet definition, peering non-transitivity, reserved IPs, NAT gateway placement, flow log limitations, MTU clamping, endpoint types, TGW route tables, and DNS attributes.

### Hallucination Patterns (`vpc-hallucination-patterns.yaml`)
8 patterns that LLMs commonly get wrong about VPC networking, including:
- Claiming security groups have deny rules (they don't)
- Applying stateful behavior to NACLs (they're stateless)
- Claiming VPC peering is transitive (it's not)
- Suggesting a "public subnet" attribute exists (it's route-table-based)
- Claiming flow logs identify which rule blocked traffic (they don't)

---

## Investigation Workflow

Each runbook follows a consistent phased structure:

### Phase 1 — Triage
Collect initial evidence using VPC APIs, route tables, security groups, NACLs, and flow logs. Classify the failure domain.

### Phase 2 — Enrich / Remediate
Deep dive using Reachability Analyzer, DNS testing, VPN/DX diagnostics, or endpoint configuration review.

### Phase 3 — Report
State root cause with evidence, severity, and recommended mitigations.

### Output Format
```yaml
root_cause: "<category> — <detail>"
evidence:
  - type: <source>
    content: "<specific finding>"
severity: CRITICAL | HIGH | MEDIUM
mitigation:
  immediate: "<action>"
  long_term: "<prevention>"
```

---

## Prerequisites

- AWS CLI or SDK access with VPC, EC2, Route 53, and CloudWatch permissions
- For flow log analysis: CloudWatch Logs or S3 access
- For Reachability Analyzer: `ec2:CreateNetworkInsightsPath` and `ec2:StartNetworkInsightsAnalysis`
- For VPN/DX: VPN and Direct Connect permissions
- For TGW: Transit Gateway permissions

---

## Usage Examples

### Can't Reach Internet from Private Subnet
```
Instances in subnet-abc can't reach the internet. They're in a private
subnet. Check NAT gateway, route tables, NACLs, and security groups.
```

### Cross-VPC Connectivity Failure
```
VPC-A (10.1.0.0/16) can't reach VPC-B (10.2.0.0/16) through Transit
Gateway. Check TGW attachments, TGW route tables, and VPC route tables.
```

### DNS Resolution Failure
```
Instances can't resolve internal.example.com. We have a private hosted
zone. Check VPC DNS settings and hosted zone associations.
```

### VPN Tunnel Down
```
Site-to-Site VPN vpn-abc has both tunnels DOWN. Check tunnel status,
IKE configuration, and customer gateway settings.
```

---

## License

MIT-0
