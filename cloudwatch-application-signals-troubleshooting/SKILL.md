---
name: cloudwatch-application-signals-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon CloudWatch Application Signals
  problems by analyzing service discovery, instrumentation, SLO configuration,
  service dependency mapping, metric collection, trace correlation, EKS/EC2 integration,
  alarm configuration, and following structured runbooks.
  Activate when: service discovery failures, instrumentation errors, SLO creation or
  evaluation issues, dependency mapping problems, metric collection failures,
  trace or log correlation errors, EKS/EC2 integration issues, alarm configuration
  problems, or the user says something is wrong with Application Signals.
compatibility: >
  Requires AWS CLI or SDK access with application-signals, cloudwatch, xray,
  logs, eks, ec2, iam, and sts permissions.
---

# CloudWatch Application Signals Diagnostics

## When to use

Any CloudWatch Application Signals investigation — service discovery, instrumentation, SLO monitoring, service dependency mapping, metric collection, trace correlation, log correlation, EKS/EC2 integration, or alarm configuration.

## Investigation workflow

### Step 1 — Collect and triage

```
aws application-signals list-service-level-objectives --query 'SloSummaries[*].{Name:Name,Arn:Arn,SliMetric:SliMetric.MetricType}'
aws application-signals list-services --start-time $(date -d '-1 hour' -u +%s) --end-time $(date -u +%s) --query 'ServiceSummaries[*].{Name:KeyAttributes.Name,Type:KeyAttributes.Type,Environment:KeyAttributes.Environment}'
aws cloudwatch list-metrics --namespace 'AWS/ApplicationSignals' --query 'Metrics[*].{MetricName:MetricName,Dimensions:Dimensions}'
```

### Step 2 — Domain deep dive

```
aws application-signals get-service-level-objective --slo-id <slo-id> --query '{Name:Name,Goal:Goal,SliConfig:SliConfig}'
aws application-signals list-service-dependencies --start-time $(date -d '-1 hour' -u +%s) --end-time $(date -u +%s) --key-attributes Name=<service-name>,Type=AWS::EC2::Instance,Environment=<env>
aws xray get-trace-summaries --start-time $(date -d '-1 hour' -u +%Y-%m-%dT%H:%M:%S) --end-time $(date -u +%Y-%m-%dT%H:%M:%S) --sampling-strategy '{"Name":"FixedRate","Value":1.0}'
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=application-signals.amazonaws.com --max-results 20
aws logs describe-log-groups --log-group-name-prefix '/aws/application-signals'
aws application-signals list-service-operations --start-time $(date -d '-1 hour' -u +%s) --end-time $(date -u +%s) --key-attributes Name=<service-name>,Type=AWS::EC2::Instance,Environment=<env>
```

Read `references/guardrails.md` before concluding on any Application Signals issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `application-signals list-services` | List discovered services |
| `application-signals list-service-level-objectives` | List SLOs |
| `application-signals get-service-level-objective` | Get SLO details |
| `application-signals list-service-dependencies` | Map service dependencies |
| `application-signals list-service-operations` | List service operations |
| `cloudwatch list-metrics --namespace AWS/ApplicationSignals` | Check Application Signals metrics |
| `xray get-trace-summaries` | Review distributed traces |

## Gotchas: CloudWatch Application Signals

- Application Signals requires the CloudWatch agent AND the AWS Distro for OpenTelemetry (ADOT) auto-instrumentation agent. Both must be running for full functionality.
- Service discovery is automatic but depends on proper instrumentation. If the ADOT agent is not injecting the correct attributes, services will not appear.
- SLOs are evaluated based on metrics in the AWS/ApplicationSignals namespace. If metrics are not flowing, SLO evaluations will show insufficient data, not failures.
- Trace correlation requires X-Ray tracing to be enabled alongside Application Signals. They are complementary, not alternatives.
- EKS integration requires the ADOT addon or DaemonSet. The CloudWatch Observability addon simplifies setup but has specific version requirements.
- EC2 integration requires manual installation of the CloudWatch agent and ADOT auto-instrumentation agent.
- Application Signals is regional. Services in different regions appear as separate services.

## Anti-hallucination rules

1. Always cite specific service names, SLO IDs, metric names, or API responses as evidence.
2. Application Signals requires BOTH CloudWatch agent AND ADOT instrumentation. Never suggest one alone is sufficient.
3. SLO insufficient data means metrics are missing, not that the SLO is healthy. Never conflate the two.
4. Service discovery depends on instrumentation attributes. Never assume services auto-discover without proper agent setup.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Discovery & Instrumentation | A1–A2 | Service discovery failures, instrumentation errors |
| B — SLOs | B1–B2 | SLO creation issues, SLO evaluation errors |
| C — Dependencies & Metrics | C1–C2 | Service dependency mapping, metric collection failures |
| D — Correlation | D1–D2 | Trace correlation issues, log correlation errors |
| E — Integration | E1–E2 | EKS integration, EC2 integration |
| F — Alarms | F1 | Alarm configuration |
| Z — Catch-All | Z1 | General troubleshooting |
