# AWS Elemental MediaStore Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Elemental MediaStore problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Elemental MediaStore — media-optimized storage for live video origination. Covers container management, policies, object operations, CORS, lifecycle policies, logging, metrics, CDN integration, origin endpoints, and throttling.

### Activate When

- Container creation failures
- Container policy errors
- Put or get object failures
- CORS configuration issues
- Lifecycle policy problems
- Access logging configuration
- Metric policy errors
- CDN integration issues
- Origin endpoint problems
- Throttling or rate limiting
- General MediaStore errors

---

## Skill Structure

```
mediastore-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-container-creation-failures.md
    ├── A2-container-policy-errors.md
    ├── B1-put-object-failures.md
    ├── B2-get-object-errors.md
    ├── C1-cors-configuration.md
    ├── C2-lifecycle-policy-issues.md
    ├── D1-access-logging.md
    ├── D2-metric-policy-errors.md
    ├── E1-cdn-integration.md
    ├── E2-origin-endpoint-issues.md
    ├── F1-throttling-issues.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Containers** | A1–A2 | Creation failures, policy errors |
| **B — Object Operations** | B1–B2 | Put failures, get errors |
| **C — Configuration** | C1–C2 | CORS, lifecycle policies |
| **D — Monitoring** | D1–D2 | Access logging, metric policies |
| **E — Integration** | E1–E2 | CDN, origin endpoints |
| **F — Performance** | F1 | Throttling |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `mediastore:*`, `mediastore-data:*`, `cloudfront:Get*`, `cloudwatch:GetMetricStatistics`, `iam:Get*` permissions

---

## License

MIT-0
