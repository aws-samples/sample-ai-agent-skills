# Amazon Route 53 Resolver Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon Route 53 Resolver problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Route 53 Resolver — resolver endpoints, forwarding rules, DNS Firewall, query logging, inbound/outbound endpoints, RAM sharing, VPC associations, and DNSSEC validation.

### Activate When

- Resolver endpoint creation failures
- Endpoint connectivity issues
- DNS forwarding rule errors
- Conditional forwarding problems
- DNS Firewall rule group issues
- DNS Firewall domain list errors
- Query logging configuration failures
- Query log analysis needs
- Inbound endpoint issues
- Outbound endpoint issues
- RAM sharing errors
- VPC association problems
- DNSSEC validation errors
- General Route 53 Resolver errors

---

## Skill Structure

```
route53-resolver-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-resolver-endpoint-creation.md
    ├── A2-resolver-endpoint-connectivity.md
    ├── B1-forwarding-rule-errors.md
    ├── B2-conditional-forwarding-issues.md
    ├── C1-dns-firewall-rule-group.md
    ├── C2-dns-firewall-domain-list.md
    ├── D1-query-logging-configuration.md
    ├── D2-query-log-analysis.md
    ├── E1-inbound-endpoint-issues.md
    ├── E2-outbound-endpoint-issues.md
    ├── F1-ram-sharing-errors.md
    ├── F2-vpc-association-issues.md
    ├── G1-dnssec-validation.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (14 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Endpoints** | A1–A2 | Endpoint creation, endpoint connectivity |
| **B — Forwarding** | B1–B2 | Forwarding rule errors, conditional forwarding |
| **C — DNS Firewall** | C1–C2 | Firewall rule groups, firewall domain lists |
| **D — Query Logging** | D1–D2 | Logging configuration, log analysis |
| **E — Direction** | E1–E2 | Inbound endpoint issues, outbound endpoint issues |
| **F — Sharing** | F1–F2 | RAM sharing errors, VPC association issues |
| **G — Security** | G1 | DNSSEC validation |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `route53resolver:*`, `ec2:Describe*`, `ram:Get*`, `logs:Describe*`, `cloudtrail:LookupEvents` permissions

---

## License

MIT-0
