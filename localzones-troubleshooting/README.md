# AWS Local Zones Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Local Zones problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Local Zones — zone enablement, subnet creation, instance launches, instance type availability, EBS volumes, networking, ELB integration, RDS, latency testing, capacity planning, and VPC extension.

### Activate When

- Local Zone enablement failures
- Subnet creation errors in Local Zones
- Instance launch failures
- Instance type not available
- EBS volume issues
- Networking configuration problems
- ELB integration errors
- RDS Local Zone issues
- Latency testing failures
- Capacity planning concerns
- VPC extension issues
- General Local Zones errors

---

## Skill Structure

```
localzones-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-local-zone-enablement.md
    ├── A2-subnet-creation-failures.md
    ├── B1-instance-launch-errors.md
    ├── B2-instance-type-availability.md
    ├── C1-ebs-volume-issues.md
    ├── C2-networking-configuration.md
    ├── D1-elb-integration.md
    ├── D2-rds-local-zone.md
    ├── E1-latency-testing.md
    ├── E2-capacity-planning.md
    ├── F1-vpc-extension-issues.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Enablement** | A1–A2 | Zone enablement, subnet creation |
| **B — Instances** | B1–B2 | Launch errors, type availability |
| **C — Storage & Network** | C1–C2 | EBS volumes, networking configuration |
| **D — Integration** | D1–D2 | ELB integration, RDS Local Zone |
| **E — Performance** | E1–E2 | Latency testing, capacity planning |
| **F — VPC** | F1 | VPC extension issues |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `ec2:Describe*`, `ec2:ModifyAvailabilityZoneGroup`, `elasticloadbalancing:Describe*`, `rds:Describe*`, `cloudwatch:GetMetricStatistics` permissions

---

## License

MIT-0
