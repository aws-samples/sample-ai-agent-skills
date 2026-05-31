# Amazon Inspector Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon Inspector problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon Inspector when the console alone isn't enough — EC2/ECR/Lambda scanning failures, coverage gaps, finding analysis, false positive management, multi-account configuration, Security Hub integration, and scan configuration.

### Activate When

- EC2 instances not being scanned
- ECR images not scanned on push
- Lambda functions not assessed
- Unexpected or missing findings
- False positive management
- Finding suppression issues
- Coverage gaps across accounts
- SSM agent not reporting to Inspector
- Security Hub findings not appearing
- EventBridge rules not triggering on findings
- Delegated admin configuration issues
- Member account enrollment problems
- Scan frequency configuration
- Exclusion rules not working
- General Inspector errors without clear symptoms

---

## Skill Structure

```
inspector-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-ec2-scanning-failures.md
    ├── A2-ecr-scanning.md
    ├── A3-lambda-scanning.md
    ├── B1-finding-analysis.md
    ├── B2-false-positives.md
    ├── B3-suppression.md
    ├── C1-coverage-gaps.md
    ├── C2-ssm-agent-requirements.md
    ├── D1-security-hub.md
    ├── D2-eventbridge.md
    ├── E1-delegated-admin.md
    ├── E2-member-accounts.md
    ├── F1-scan-frequency.md
    ├── F2-exclusion-rules.md
    ├── Z1-general-troubleshooting.md
    ├── inspector-guardrails.md
    └── inspector-hallucination-patterns.yaml
```

---

## Runbook Library (20 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Scanning** | A1–A3 | EC2 scanning failures, ECR scanning, Lambda scanning |
| **B — Findings** | B1–B3 | Finding analysis, false positives, suppression |
| **C — Coverage** | C1–C2 | Coverage gaps, SSM agent requirements |
| **D — Integration** | D1–D2 | Security Hub, EventBridge |
| **E — Multi-Account** | E1–E2 | Delegated admin, member accounts |
| **F — Configuration** | F1–F2 | Scan frequency, exclusion rules |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/inspector-guardrails.md` covering Inspector v2 vs Classic distinction, SSM agent requirement for EC2, ECR automatic scanning behavior, Lambda code vs standard scanning, regional finding scope, suppression vs remediation, pricing model, coverage gap analysis, delegated admin limitations, finding severity levels, scan trigger mechanisms, and resource tag-based filtering.

---

## Investigation Workflow

1. **Triage** — Check account status, coverage statistics, active findings, scan types enabled
2. **Deep Dive** — Examine specific resource coverage, SSM agent status, finding details
3. **Detailed** — Review member accounts, delegated admin, CloudTrail events, suppression filters

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `inspector2:*`, `ssm:DescribeInstanceInformation`, `ecr:DescribeRepositories`, `lambda:ListFunctions`, `securityhub:GetFindings`, `events:ListRules`
- Inspector v2 enabled in the account/region
- SSM agent installed and running on EC2 instances for vulnerability scanning

---

## License

MIT-0
