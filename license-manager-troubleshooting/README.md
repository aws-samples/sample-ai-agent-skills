# AWS License Manager Diagnostics Skill

Agent skill for investigating and troubleshooting AWS License Manager problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS License Manager — managing software licenses across AWS and on-premises environments including license configurations, grants, dedicated hosts, subscriptions, and cross-account sharing.

### Activate When

- License configuration errors
- License rule violations
- Grant creation failures
- Grant acceptance issues
- Host resource group errors
- Dedicated host management problems
- Linux subscription discovery issues
- User subscription problems
- Cross-account sharing failures
- Organization settings issues
- Automated discovery problems
- General License Manager errors

---

## Skill Structure

```
license-manager-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-license-configuration-errors.md
    ├── A2-license-rule-violations.md
    ├── B1-grant-creation-failures.md
    ├── B2-grant-acceptance-issues.md
    ├── C1-host-resource-group-errors.md
    ├── C2-dedicated-host-management.md
    ├── D1-linux-subscription-discovery.md
    ├── D2-user-subscription-issues.md
    ├── E1-cross-account-sharing.md
    ├── E2-organization-settings.md
    ├── F1-automated-discovery.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — License Config** | A1–A2 | Configuration errors, rule violations |
| **B — Grants** | B1–B2 | Grant creation, grant acceptance |
| **C — Dedicated Hosts** | C1–C2 | Host resource groups, host management |
| **D — Subscriptions** | D1–D2 | Linux subscriptions, user subscriptions |
| **E — Cross-Account** | E1–E2 | Cross-account sharing, org settings |
| **F — Discovery** | F1 | Automated discovery |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `license-manager:*`, `ec2:Describe*`, `iam:Get*`, `organizations:Describe*`, `ram:Get*`, `ssm:Describe*` permissions

---

## License

MIT-0
