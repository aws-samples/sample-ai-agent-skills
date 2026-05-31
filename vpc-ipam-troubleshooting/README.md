# VPC IPAM Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon VPC IPAM problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for VPC IPAM — IPAM creation, scope configuration, pool management, CIDR provisioning, allocation failures, overlap detection, organization integration, cross-region configuration, BYOIP, public IP insights, and compliance monitoring.

### Activate When

- IPAM creation failures
- Scope configuration issues
- Pool creation errors
- CIDR provisioning problems
- Allocation failures
- CIDR overlap detection
- Organization integration issues
- Cross-region configuration problems
- BYOIP provisioning errors
- Public IP insights issues
- Compliance monitoring failures
- General VPC IPAM errors

---

## Skill Structure

```
vpc-ipam-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-ipam-creation-failures.md
    ├── A2-ipam-scope-configuration.md
    ├── B1-pool-creation-errors.md
    ├── B2-pool-cidr-provisioning.md
    ├── C1-allocation-failures.md
    ├── C2-cidr-overlap-detection.md
    ├── D1-organization-integration.md
    ├── D2-cross-region-configuration.md
    ├── E1-byoip-provisioning.md
    ├── E2-public-ip-insights.md
    ├── F1-compliance-monitoring.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — IPAM Setup** | A1–A2 | Creation failures, scope configuration |
| **B — Pools** | B1–B2 | Pool creation, CIDR provisioning |
| **C — Allocations** | C1–C2 | Allocation failures, overlap detection |
| **D — Organization** | D1–D2 | Organization integration, cross-region |
| **E — BYOIP & Public IP** | E1–E2 | BYOIP provisioning, public IP insights |
| **F — Compliance** | F1 | Compliance monitoring |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `ec2:*Ipam*`, `ec2:DescribeVpcs`, `organizations:Describe*`, `ram:*`, `iam:Get*`, `cloudtrail:LookupEvents` permissions

---

## License

MIT-0
