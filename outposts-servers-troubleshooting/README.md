# AWS Outposts Servers Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Outposts Servers (1U/2U form factor) problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Outposts Servers — server ordering, activation, EC2 launches, EBS local volumes, network connectivity, service link, capacity, updates, monitoring, and rack-vs-server differences.

### Activate When

- Server ordering or delivery issues
- Server activation failures
- EC2 instance launch errors on Outposts server
- EBS local volume problems
- Network connectivity issues at edge site
- Service link failures to parent Region
- Capacity constraints on single server
- Software update failures
- Monitoring or alerting gaps
- Tag management issues
- Questions about rack vs server differences

---

## Skill Structure

```
outposts-servers-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-server-ordering-issues.md
    ├── A2-server-activation-failures.md
    ├── B1-ec2-launch-errors.md
    ├── B2-ebs-local-volume-issues.md
    ├── C1-network-connectivity.md
    ├── C2-service-link-failures.md
    ├── D1-capacity-management.md
    ├── D2-software-updates.md
    ├── E1-monitoring-configuration.md
    ├── E2-tag-management.md
    ├── F1-rack-vs-server-differences.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Ordering & Setup** | A1–A2 | Ordering, activation |
| **B — Compute & Storage** | B1–B2 | EC2 launches, EBS volumes |
| **C — Networking** | C1–C2 | Connectivity, service link |
| **D — Operations** | D1–D2 | Capacity, updates |
| **E — Observability** | E1–E2 | Monitoring, tagging |
| **F — Architecture** | F1 | Rack vs server |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `outposts:*`, `ec2:Describe*`, `ebs:*`, `cloudwatch:GetMetricStatistics`, `ssm:*`, `iam:Get*` permissions
- Physical site access may be required for hardware-level troubleshooting

---

## License

MIT-0
