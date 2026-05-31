# AWS Transit Gateway Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Transit Gateway problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Transit Gateway when the console alone isn't enough — TGW creation, route table configuration, VPC/VPN/DX attachments, inter-region peering, route propagation, blackhole routes, multicast domains, and cross-AZ connectivity.

### Activate When

- TGW creation failures or configuration issues
- Route table misconfiguration or missing routes
- Attachment failures (VPC, VPN, DX, peering)
- VPC attachment subnet routing problems
- Cross-AZ connectivity issues through TGW
- Inter-region peering not working
- Peering route propagation failures
- VPN attachment or tunnel issues
- BGP session failures over TGW
- Direct Connect gateway attachment problems
- Transit VIF configuration errors
- Route propagation not working as expected
- Blackhole routes blocking traffic silently
- Route table association errors
- Multicast domain configuration issues
- Multicast group membership problems
- General Transit Gateway errors without clear symptoms

---

## Skill Structure

```
transitgateway-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-creation-failures.md
    ├── A2-route-table-issues.md
    ├── A3-attachment-failures.md
    ├── B1-vpc-attachment.md
    ├── B2-subnet-routing.md
    ├── B3-cross-az.md
    ├── C1-inter-region-peering.md
    ├── C2-peering-route-propagation.md
    ├── D1-vpn-attachment.md
    ├── D2-bgp-over-tgw.md
    ├── E1-dx-gateway-attachment.md
    ├── E2-transit-vif.md
    ├── F1-route-propagation.md
    ├── F2-blackhole-routes.md
    ├── F3-route-table-association.md
    ├── G1-multicast-domain.md
    ├── G2-group-membership.md
    ├── Z1-general-troubleshooting.md
    ├── transitgateway-guardrails.md
    └── transitgateway-hallucination-patterns.yaml
```

---

## Runbook Library (24 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — TGW** | A1–A3 | Creation failures, route table issues, attachment failures |
| **B — VPC** | B1–B3 | VPC attachment, subnet routing, cross-AZ |
| **C — Peering** | C1–C2 | Inter-region peering, peering route propagation |
| **D — VPN** | D1–D2 | VPN attachment, BGP over TGW |
| **E — Direct Connect** | E1–E2 | DX gateway attachment, transit VIF |
| **F — Routing** | F1–F3 | Route propagation, blackhole routes, route table association |
| **G — Multicast** | G1–G2 | Multicast domain, group membership |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/transitgateway-guardrails.md` covering TGW vs VPC route table distinction, subnet-per-AZ requirement, route propagation vs static routes, blackhole route detection, inter-region peering limitations, one-association-per-attachment rule, appliance mode for stateful inspection, MTU considerations, cross-account sharing via RAM, default route table behavior, multicast creation-time requirement, and attachment lifecycle states.

---

## Investigation Workflow

1. **Triage** — Check TGW status, list attachments, examine route tables, search for blackhole routes
2. **Deep Dive** — Examine VPC attachments, peering attachments, route table associations and propagations
3. **Detailed** — Review CloudTrail events, multicast domains, CloudWatch metrics

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `ec2:DescribeTransitGateways`, `ec2:DescribeTransitGatewayAttachments`, `ec2:DescribeTransitGatewayRouteTables`, `ec2:SearchTransitGatewayRoutes`, `ec2:DescribeTransitGatewayVpcAttachments`, `cloudtrail:LookupEvents`
- VPC Flow Logs enabled for traffic analysis (optional but recommended)
- RAM permissions for cross-account TGW sharing issues

---

## Usage Examples

```
# Check TGW status
aws ec2 describe-transit-gateways --transit-gateway-ids tgw-12345678

# List all attachments
aws ec2 describe-transit-gateway-attachments --filters Name=transit-gateway-id,Values=tgw-12345678

# Search for blackhole routes
aws ec2 search-transit-gateway-routes --transit-gateway-route-table-id tgw-rtb-12345678 --filters Name=state,Values=blackhole

# Check route propagation
aws ec2 get-transit-gateway-route-table-propagations --transit-gateway-route-table-id tgw-rtb-12345678
```

---

## License

MIT-0
