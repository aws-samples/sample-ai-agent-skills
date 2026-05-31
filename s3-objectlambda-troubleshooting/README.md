# S3 Object Lambda Diagnostics Skill

Agent skill for investigating and troubleshooting S3 Object Lambda problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for S3 Object Lambda — access point creation, configuration, Lambda invocation errors, transformation failures, permissions, performance, range/HEAD requests, and CloudFront integration.

### Activate When

- Object Lambda access point creation failures
- Lambda invocation errors during object retrieval
- Transformation failures or incorrect output
- Permission errors accessing OLAPs
- Performance degradation or timeout errors
- Range GET or part-number request issues
- HEAD object request failures
- CloudFront integration problems
- General S3 Object Lambda errors

---

## Skill Structure

```
s3-objectlambda-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-access-point-creation.md
    ├── A2-access-point-configuration.md
    ├── B1-lambda-invocation-errors.md
    ├── B2-transformation-failures.md
    ├── C1-permission-errors.md
    ├── C2-policy-configuration.md
    ├── D1-performance-issues.md
    ├── D2-timeout-errors.md
    ├── E1-range-get-issues.md
    ├── E2-head-object-issues.md
    ├── F1-cloudfront-integration.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Access Points** | A1–A2 | Creation, configuration |
| **B — Lambda Execution** | B1–B2 | Invocation errors, transformation failures |
| **C — Permissions** | C1–C2 | Permission errors, policy configuration |
| **D — Performance** | D1–D2 | Performance issues, timeout errors |
| **E — Request Handling** | E1–E2 | Range GET issues, HEAD object issues |
| **F — Integration** | F1 | CloudFront integration |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `s3:*`, `s3-object-lambda:*`, `lambda:*`, `iam:Get*`, `cloudfront:Get*`, `cloudwatch:GetMetricStatistics`, `logs:FilterLogEvents` permissions

---

## License

MIT-0
