# ECS Anywhere Diagnostics Skill

Agent skill for investigating and troubleshooting ECS Anywhere problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for ECS Anywhere — external instance registration, SSM agent issues, task placement, container runtime, networking, service discovery, IAM roles, credentials, monitoring, logging, and capacity providers.

### Activate When

- External instance registration failures
- SSM Agent connectivity issues
- Task placement failures on external instances
- Container runtime errors
- Networking or service discovery problems
- IAM role or credential issues
- Monitoring or log collection gaps
- Capacity provider configuration errors
- General ECS Anywhere errors

---

## Skill Structure

```
ecs-anywhere-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-external-instance-registration.md
    ├── A2-ssm-agent-issues.md
    ├── B1-task-placement-failures.md
    ├── B2-container-runtime-errors.md
    ├── C1-networking-issues.md
    ├── C2-service-discovery.md
    ├── D1-iam-role-configuration.md
    ├── D2-credential-management.md
    ├── E1-monitoring-gaps.md
    ├── E2-log-collection.md
    ├── F1-capacity-provider.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Registration** | A1–A2 | Instance registration, SSM agent |
| **B — Task Execution** | B1–B2 | Placement failures, container runtime |
| **C — Networking** | C1–C2 | Networking, service discovery |
| **D — IAM & Credentials** | D1–D2 | IAM roles, credential management |
| **E — Observability** | E1–E2 | Monitoring, log collection |
| **F — Capacity** | F1 | Capacity provider |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `ecs:*`, `ssm:*`, `iam:Get*`, `cloudwatch:GetMetricStatistics`, `logs:FilterLogEvents` permissions
- External instances: SSM Agent, Docker or containerd, ECS Agent
- Network connectivity from external instances to AWS endpoints

---

## License

MIT-0
