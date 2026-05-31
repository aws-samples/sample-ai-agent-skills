# AWS Service Quotas Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Service Quotas problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Service Quotas when the console alone isn't enough — quota lookups, increase requests, utilization monitoring, organization-wide quota management, service-specific limits, and automated quota management.

### Activate When

- Hitting service limits or throttling
- Quota increase request issues
- CloudWatch quota alarm configuration
- Utilization monitoring setup
- Organization-wide quota management
- EC2 instance vCPU limits
- Lambda concurrency limits
- VPC resource limits
- Automated quota management
- General quota errors without clear symptoms

---

## Skill Structure

```
servicequotas-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-quota-lookup.md
    ├── A2-quota-increase-requests.md
    ├── A3-applied-vs-default.md
    ├── B1-cloudwatch-quota-alarms.md
    ├── B2-utilization-monitoring.md
    ├── C1-organization-quotas.md
    ├── C2-quota-templates.md
    ├── D1-ec2-limits.md
    ├── D2-lambda-limits.md
    ├── D3-vpc-limits.md
    ├── E1-quota-request-automation.md
    ├── E2-service-quotas-api.md
    ├── Z1-general-troubleshooting.md
    ├── servicequotas-guardrails.md
    └── servicequotas-hallucination-patterns.yaml
```

---

## Runbook Library (16 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Quotas** | A1–A3 | Quota lookup, quota increase requests, applied vs default |
| **B — Alarms** | B1–B2 | CloudWatch quota alarms, utilization monitoring |
| **C — Organization** | C1–C2 | Organization quotas, quota templates |
| **D — Service-Specific** | D1–D3 | EC2 limits, Lambda limits, VPC limits |
| **E — Automation** | E1–E2 | Quota request automation, Service Quotas API |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/servicequotas-guardrails.md` covering adjustable vs non-adjustable quotas, applied vs default values, increase request timelines, CloudWatch Usage metric availability, organization template scope, regional vs global quotas, API service coverage, utilization spikes, EC2 vCPU-based limits, Lambda concurrency model, VPC resource limits, and quota increase approval factors.

---

## Investigation Workflow

1. **Triage** — Check current quota values, utilization, recent increase requests
2. **Deep Dive** — Examine specific quota details, CloudWatch metrics, request history
3. **Detailed** — Review organization templates, automation options, cost implications

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `servicequotas:*`, `cloudwatch:GetMetricStatistics`, `ec2:DescribeAccountAttributes`, `lambda:GetAccountSettings`
- Service Quotas console access for visual quota management

---

## License

MIT-0
