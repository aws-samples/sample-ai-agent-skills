# AWS Verified Access Advanced Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Verified Access advanced problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Verified Access — instance creation, trust provider configuration, Cedar policy debugging, endpoint groups, device and user trust evaluation, access log analysis, WAF integration, and multi-trust-provider patterns.

### Activate When

- Verified Access instance creation failures
- Trust provider configuration issues
- Cedar policy syntax or evaluation errors
- Endpoint group or endpoint connectivity problems
- Device trust integration failures
- User trust evaluation issues
- Access log analysis needs
- WAF integration with Verified Access
- Multi-trust-provider configuration
- General zero-trust access issues

---

## Skill Structure

```
verified-access-advanced-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-instance-creation-failures.md
    ├── A2-trust-provider-advanced.md
    ├── B1-cedar-policy-debugging.md
    ├── B2-policy-evaluation-errors.md
    ├── C1-endpoint-group-configuration.md
    ├── C2-endpoint-connectivity-advanced.md
    ├── D1-device-trust-integration.md
    ├── D2-user-trust-evaluation.md
    ├── E1-access-log-analysis.md
    ├── E2-waf-integration.md
    ├── F1-multi-trust-provider.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Instance & Trust** | A1–A2 | Instance creation, trust providers |
| **B — Policies** | B1–B2 | Cedar debugging, evaluation errors |
| **C — Endpoints** | C1–C2 | Endpoint groups, connectivity |
| **D — Trust Evaluation** | D1–D2 | Device trust, user trust |
| **E — Observability & WAF** | E1–E2 | Access logs, WAF integration |
| **F — Advanced** | F1 | Multi-trust-provider |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `ec2:DescribeVerifiedAccess*`, `ec2:CreateVerifiedAccess*`, `ec2:ModifyVerifiedAccess*`, `logs:*`, `wafv2:*`, `sso:*`, `iam:Get*` permissions

---

## License

MIT-0
