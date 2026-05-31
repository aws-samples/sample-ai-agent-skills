# Edge Functions Diagnostics Skill (CloudFront Functions & Lambda@Edge)

Agent skill for investigating and troubleshooting CloudFront Functions and Lambda@Edge problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for CloudFront Functions and Lambda@Edge — function creation, publishing, deployment, replication, viewer/origin event errors, timeouts, memory limits, testing, logging, and cache behavior association.

### Activate When

- CloudFront Function creation or publish failures
- Lambda@Edge deployment or replication issues
- Viewer request/response errors
- Origin request/response errors
- Function timeout or memory limit errors
- Testing failures in development or staging
- Logging and debugging issues
- Cache behavior association problems
- General edge function errors

---

## Skill Structure

```
edge-functions-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-cloudfront-function-creation.md
    ├── A2-function-publish-errors.md
    ├── B1-lambda-edge-deployment.md
    ├── B2-lambda-edge-replication.md
    ├── C1-viewer-request-errors.md
    ├── C2-viewer-response-errors.md
    ├── D1-origin-request-errors.md
    ├── D2-origin-response-errors.md
    ├── E1-function-timeout.md
    ├── E2-function-memory-limits.md
    ├── F1-testing-failures.md
    ├── F2-logging-issues.md
    ├── G1-cache-behavior-association.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (14 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — CloudFront Functions** | A1–A2 | Creation, publish errors |
| **B — Lambda@Edge Deployment** | B1–B2 | Deployment, replication |
| **C — Viewer Events** | C1–C2 | Viewer request, viewer response |
| **D — Origin Events** | D1–D2 | Origin request, origin response |
| **E — Resource Limits** | E1–E2 | Timeout, memory limits |
| **F — Development** | F1–F2 | Testing, logging |
| **G — Association** | G1 | Cache behavior association |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `cloudfront:*`, `lambda:*`, `iam:Get*`, `cloudwatch:GetMetricStatistics`, `logs:FilterLogEvents` permissions
- Lambda@Edge functions must be in us-east-1

---

## License

MIT-0
