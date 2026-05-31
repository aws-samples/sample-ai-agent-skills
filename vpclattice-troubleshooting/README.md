# Amazon VPC Lattice Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon VPC Lattice problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon VPC Lattice — service network management, service creation and configuration, target group health and registration, listener rules and routing, auth policy evaluation, cross-account sharing via RAM, DNS resolution, health checks, access logging, and VPC association issues.

### Activate When

- Service network creation or update failures
- Service registration or configuration issues
- Target group targets unhealthy or draining
- Listener rules not matching expected traffic
- Auth policy denying legitimate requests
- Cross-account sharing not working via RAM
- DNS resolution failures for Lattice services
- Health check failures for targets
- Access logs missing or incomplete
- VPC association errors or pending state
- General VPC Lattice errors without clear symptoms

---

## Skill Structure

```
vpclattice-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-service-network-creation.md
    ├── A2-service-network-config.md
    ├── B1-service-creation.md
    ├── B2-listener-config.md
    ├── C1-target-registration.md
    ├── C2-target-health.md
    ├── D1-rule-matching.md
    ├── D2-routing-issues.md
    ├── E1-auth-policy-evaluation.md
    ├── E2-cross-level-auth.md
    ├── F1-ram-sharing.md
    ├── F2-consumer-association.md
    ├── G1-dns-resolution.md
    ├── G2-health-checks.md
    ├── H1-access-logs.md
    ├── H2-vpc-association.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (14 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Service Network** | A1–A2 | Creation failures, configuration |
| **B — Services** | B1–B2 | Service creation, listener config |
| **C — Target Groups** | C1–C2 | Target registration, health issues |
| **D — Listener Rules** | D1–D2 | Rule matching, routing issues |
| **E — Auth Policies** | E1–E2 | Policy evaluation, cross-level auth |
| **F — Cross-Account** | F1–F2 | RAM sharing, consumer association |
| **G — DNS & Health** | G1–G2 | DNS resolution, health checks |
| **H — Logging & VPC** | H1–H2 | Access logs, VPC association |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `vpc-lattice:Get*`, `vpc-lattice:List*`, `ram:Get*`, `logs:FilterLogEvents`, `cloudtrail:LookupEvents`, `cloudwatch:GetMetricStatistics`

---

## License

MIT-0
