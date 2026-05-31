# AWS X-Ray Diagnostics Skill

Agent skill for investigating and troubleshooting AWS X-Ray distributed tracing problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS X-Ray when the console alone isn't enough — missing traces, sampling configuration, SDK instrumentation, daemon issues, Lambda/ECS/API Gateway integration, service map gaps, trace analysis, group filters, insights, IAM permissions, and encryption.

### Activate When

- Traces not appearing in X-Ray console
- Incomplete traces with missing segments or subsegments
- Sampling rules not applying as expected
- SDK instrumentation errors or missing downstream calls
- X-Ray daemon not forwarding segments
- Segment document errors or rejections
- Lambda function traces missing
- ECS/EKS container traces incomplete
- API Gateway tracing not working
- Service map showing gaps or missing services
- Trace analysis returning unexpected results
- Group filter expressions not matching
- Anomaly detection not generating insights
- Latency analysis issues
- IAM permission errors for X-Ray
- Encryption configuration problems
- General X-Ray errors without clear symptoms

---

## Skill Structure

```
xray-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-missing-traces.md
    ├── A2-incomplete-traces.md
    ├── A3-sampling-rules.md
    ├── B1-sdk-instrumentation.md
    ├── B2-daemon-configuration.md
    ├── B3-segment-errors.md
    ├── C1-lambda-tracing.md
    ├── C2-ecs-eks-tracing.md
    ├── C3-apigateway-tracing.md
    ├── D1-service-map-issues.md
    ├── D2-trace-analysis.md
    ├── E1-group-configuration.md
    ├── E2-filter-expressions.md
    ├── F1-anomaly-detection.md
    ├── F2-latency-analysis.md
    ├── G1-iam-permissions.md
    ├── G2-encryption.md
    ├── Z1-general-troubleshooting.md
    ├── xray-guardrails.md
    └── xray-hallucination-patterns.yaml
```

---

## Runbook Library (22 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Tracing** | A1–A3 | Missing traces, incomplete traces, sampling rules |
| **B — SDK** | B1–B3 | SDK instrumentation, daemon configuration, segment errors |
| **C — Integration** | C1–C3 | Lambda tracing, ECS/EKS tracing, API Gateway tracing |
| **D — Analysis** | D1–D2 | Service map issues, trace analysis |
| **E — Groups** | E1–E2 | Group configuration, filter expressions |
| **F — Insights** | F1–F2 | Anomaly detection, latency analysis |
| **G — Security** | G1–G2 | IAM permissions, encryption |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/xray-guardrails.md` covering daemon requirement for non-Lambda services, sampling rule priority evaluation, segment vs subsegment distinction, trace ID propagation requirements, 64KB segment document limit, 30-day trace retention, Lambda automatic integration, ECS sidecar daemon requirement, SDK instrumentation necessity, X-Ray SDK vs OpenTelemetry, service map time range, and sampling statistics interpretation.

---

## Investigation Workflow

1. **Triage** — Check service graph, trace summaries, sampling rules, groups
2. **Deep Dive** — Examine full traces, sampling statistics, insights, encryption
3. **Detailed** — Review CloudTrail events, IAM permissions, daemon logs

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `xray:GetServiceGraph`, `xray:GetTraceSummaries`, `xray:BatchGetTraces`, `xray:GetSamplingRules`, `xray:GetGroups`, `xray:GetInsightSummaries`, `xray:GetEncryptionConfig`
- Application-level access for SDK instrumentation issues
- CloudWatch Logs access for daemon log analysis

---

## Usage Examples

```
# Get service graph for last hour
aws xray get-service-graph --start-time $(date -u -d '1 hour ago' +%s) --end-time $(date -u +%s)

# Get trace summaries
aws xray get-trace-summaries --start-time $(date -u -d '1 hour ago' +%s) --end-time $(date -u +%s)

# Check sampling rules
aws xray get-sampling-rules

# Get full trace details
aws xray batch-get-traces --trace-ids 1-12345678-abcdef

# Check encryption config
aws xray get-encryption-config
```

---

## License

MIT-0
