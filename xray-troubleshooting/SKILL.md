---
name: xray-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS X-Ray distributed tracing
  problems by analyzing missing traces, incomplete traces, sampling rules,
  SDK instrumentation, daemon configuration, segment errors, Lambda tracing,
  ECS/EKS tracing, API Gateway tracing, service map issues, trace analysis,
  group configuration, filter expressions, anomaly detection, latency analysis,
  IAM permissions, encryption settings, and following structured runbooks.
  Activate when: traces not appearing, incomplete trace segments, sampling
  rules not applying, SDK instrumentation errors, daemon not forwarding,
  segment document errors, Lambda traces missing, ECS/EKS traces incomplete,
  API Gateway tracing issues, service map gaps, trace analysis problems,
  group filter not matching, filter expression errors, anomaly detection
  not working, latency analysis issues, IAM permission errors, encryption
  configuration problems, or the user says something is wrong with X-Ray
  without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with X-Ray, Lambda, ECS, API Gateway, IAM,
  CloudWatch, and CloudTrail permissions. Some operations require application-level
  SDK instrumentation access.
---

# AWS X-Ray Diagnostics

## When to use

Any AWS X-Ray investigation where the console alone is insufficient — missing traces, sampling configuration, SDK instrumentation, daemon issues, service map gaps, Lambda/ECS/API Gateway integration, group filters, insights, IAM permissions, or encryption troubleshooting.

## Investigation workflow

### Step 1 — Collect and triage

```
aws xray get-service-graph --start-time <start> --end-time <end>
aws xray get-trace-summaries --start-time <start> --end-time <end> --sampling-strategy '{"Name":"FixedRate","Value":1.0}'
aws xray get-sampling-rules
aws xray get-groups
```

### Step 2 — Domain deep dive

```
aws xray batch-get-traces --trace-ids <trace-id>
aws xray get-sampling-statistic-summaries
aws xray get-insight-summaries --start-time <start> --end-time <end> --states ACTIVE
aws xray get-encryption-config
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=xray.amazonaws.com --max-results 20
aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names xray:PutTraceSegments xray:PutTelemetryRecords
aws logs filter-log-events --log-group-name <daemon-log-group> --filter-pattern "ERROR"
```

Read `references/xray-guardrails.md` before concluding on any X-Ray issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `get-service-graph` | Visualize service dependencies and latency |
| `get-trace-summaries` | Find traces matching criteria |
| `batch-get-traces` | Get full trace details with segments |
| `get-sampling-rules` | Check sampling configuration |
| `get-sampling-statistic-summaries` | Check actual sampling rates |
| `get-groups` | List trace groups and filters |
| `get-insight-summaries` | Check active insights and anomalies |
| `get-encryption-config` | Check encryption settings |
| CloudTrail | Audit X-Ray API calls |
| CloudWatch Logs | Check daemon and application logs |

## Gotchas: AWS X-Ray

- X-Ray daemon must be running for non-Lambda services. Lambda has built-in X-Ray integration and does not need a daemon. For ECS, the daemon runs as a sidecar container. For EC2, it runs as a system service. The daemon listens on UDP port 2000.
- Sampling rules control what percentage of requests are traced. Rules are evaluated in priority order (lowest number first). The default rule catches everything not matched by custom rules. Misconfigured priorities cause unexpected sampling behavior.
- Segments vs subsegments: segments represent a service processing a request. Subsegments represent downstream calls (AWS SDK, HTTP, SQL). Missing subsegments usually mean the SDK is not instrumented for those calls.
- Trace ID propagation across services requires the X-Amzn-Trace-Id header. If a service does not forward this header, downstream traces appear as separate traces, not connected to the original.
- 64KB segment document limit. Traces with too many subsegments or large metadata/annotations are truncated or rejected. Keep annotations small and use metadata sparingly.
- Traces are retained for 30 days. Historical traces beyond retention are permanently lost. Use X-Ray groups and CloudWatch metrics for long-term analysis.
- X-Ray SDK vs OpenTelemetry SDK: both can send traces to X-Ray. OpenTelemetry is the newer standard. The X-Ray SDK is AWS-specific. They use different instrumentation approaches but both work with the X-Ray backend.
- Service map shows last 6 hours by default. Adjust the time range to see historical service maps. The service map is generated from trace data, so missing traces mean incomplete maps.

## Anti-hallucination rules

1. Always cite specific trace IDs, segment IDs, or API responses as evidence.
2. SDK instrumentation is required for downstream tracing. Never claim automatic full-stack tracing.
3. The daemon is required for non-Lambda services. Never skip daemon verification.
4. Sampling rules have priority order. Never ignore rule priority when diagnosing sampling issues.
5. Trace retention is 30 days. Never promise access to older traces.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 22 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Tracing | A1–A3 | Missing traces, incomplete traces, sampling rules |
| B — SDK | B1–B3 | SDK instrumentation, daemon configuration, segment errors |
| C — Integration | C1–C3 | Lambda tracing, ECS/EKS tracing, API Gateway tracing |
| D — Analysis | D1–D2 | Service map issues, trace analysis |
| E — Groups | E1–E2 | Group configuration, filter expressions |
| F — Insights | F1–F2 | Anomaly detection, latency analysis |
| G — Security | G1–G2 | IAM permissions, encryption |
| Z — Catch-All | Z1 | General troubleshooting |
