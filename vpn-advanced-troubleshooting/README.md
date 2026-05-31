# AWS Site-to-Site VPN Advanced Diagnostics Skill

Agent skill for investigating and troubleshooting advanced AWS Site-to-Site VPN problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for advanced AWS Site-to-Site VPN — accelerated VPN, ECMP, certificate-based authentication, private IP VPN, CloudWatch monitoring, log analysis, BGP route propagation, static route failover, tunnel health checks, DPD timeouts, and multi-tunnel load balancing.

### Activate When

- Accelerated VPN performance or creation issues
- ECMP configuration or load balancing problems
- Certificate-based authentication failures
- Private IP VPN over Direct Connect issues
- VPN CloudWatch monitoring gaps
- VPN log analysis needs
- BGP route propagation delays or failures
- Static route failover not working
- Tunnel health check failures
- DPD timeout or tunnel flapping issues
- Multi-tunnel load balancing problems
- General advanced VPN errors

---

## Skill Structure

```
vpn-advanced-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-accelerated-vpn-issues.md
    ├── A2-vpn-ecmp-configuration.md
    ├── B1-certificate-based-auth.md
    ├── B2-private-ip-vpn.md
    ├── C1-vpn-cloudwatch-monitoring.md
    ├── C2-vpn-log-analysis.md
    ├── D1-bgp-route-propagation.md
    ├── D2-static-route-failover.md
    ├── E1-tunnel-health-checks.md
    ├── E2-dpd-timeout-issues.md
    ├── F1-multi-tunnel-load-balancing.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Accelerated & ECMP** | A1–A2 | Accelerated VPN, ECMP configuration |
| **B — Authentication** | B1–B2 | Certificate-based auth, private IP VPN |
| **C — Monitoring** | C1–C2 | CloudWatch monitoring, VPN log analysis |
| **D — Routing** | D1–D2 | BGP route propagation, static route failover |
| **E — Tunnel Health** | E1–E2 | Tunnel health checks, DPD timeouts |
| **F — Load Balancing** | F1 | Multi-tunnel load balancing |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `ec2:DescribeVpn*`, `ec2:DescribeCustomerGateways`, `ec2:DescribeTransitGateway*`, `cloudwatch:GetMetricData`, `logs:FilterLogEvents`, `acm-pca:Describe*` permissions

---

## License

MIT-0
