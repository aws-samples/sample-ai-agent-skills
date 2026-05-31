# AWS Network Manager / Cloud WAN Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Network Manager and Cloud WAN problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Network Manager and Cloud WAN — global networks, core network policies, attachments, peering, routing, segments, transit gateway registration, VPN attachments, telemetry, and policy versioning.

### Activate When

- Global network creation failures
- Core network policy errors or validation failures
- Attachment creation or state failures
- Peering connection issues
- Route analysis errors or missing routes
- Routing policy misconfiguration
- Segment configuration problems
- Network function group issues
- Transit gateway registration failures
- Site-to-site VPN attachment problems
- Telemetry data gaps
- CloudWatch integration issues
- Policy version management errors
- General Network Manager or Cloud WAN errors

---

## Skill Structure

```
network-manager-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-global-network-creation.md
    ├── A2-core-network-policy-errors.md
    ├── B1-attachment-failures.md
    ├── B2-peering-issues.md
    ├── C1-route-analysis-errors.md
    ├── C2-routing-policy-issues.md
    ├── D1-segment-configuration.md
    ├── D2-network-function-groups.md
    ├── E1-transit-gateway-registration.md
    ├── E2-site-to-site-vpn-attachment.md
    ├── F1-telemetry-issues.md
    ├── F2-cloudwatch-integration.md
    ├── G1-policy-version-management.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (14 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Global Network** | A1–A2 | Creation, core network policy errors |
| **B — Attachments** | B1–B2 | Attachment failures, peering issues |
| **C — Routing** | C1–C2 | Route analysis errors, routing policy issues |
| **D — Segments** | D1–D2 | Segment configuration, network function groups |
| **E — Transit Gateway** | E1–E2 | TGW registration, site-to-site VPN attachment |
| **F — Telemetry** | F1–F2 | Telemetry issues, CloudWatch integration |
| **G — Policy Management** | G1 | Policy version management |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `networkmanager:*`, `ec2:DescribeTransitGateways`, `ec2:DescribeVpnConnections`, `cloudwatch:GetMetricData`, `cloudtrail:LookupEvents` permissions

---

## License

MIT-0
