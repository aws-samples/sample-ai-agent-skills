---
name: amp-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon Managed Service
  for Prometheus (AMP) problems by analyzing workspace configurations,
  remote write failures, query performance, alerting/ruler issues,
  ingestion problems, Prometheus-compatible API issues, IAM/SigV4
  authentication, and collector configuration. Activate when: workspace
  creation failures, remote write errors, SigV4 authentication failures,
  ingestion rate limiting, high cardinality issues, query timeouts,
  PromQL errors, rule evaluation failures, SNS alert integration issues,
  ADOT collector problems, Prometheus server remote write configuration,
  EKS add-on issues, Grafana data source setup, VPC endpoint access,
  or the user says something is wrong with AMP without naming specific
  symptoms.
compatibility: >
  Requires AWS CLI with amp and cloudwatch permissions, promtool
  (optional), Prometheus or Grafana client for query validation,
  CloudTrail permissions. Optional: ADOT Collector CLI, kubectl
  (for EKS-based collectors).
---

# AMP Diagnostics

## When to use

Any AMP investigation where the console alone is insufficient — workspace lifecycle issues, remote write ingestion failures, SigV4 authentication errors, query performance problems, alerting/ruler configuration, collector troubleshooting, or integration with Grafana and third-party tools.

## Investigation workflow

### Step 1 — Collect and triage

```
# List all workspaces and their status
aws amp list-workspaces --query 'workspaces[*].{id:workspaceId,alias:alias,status:status.statusCode}'

# Describe specific workspace
aws amp describe-workspace --workspace-id <workspace-id>

# Check CloudWatch metrics for AMP workspace
# Key metrics: ActiveSeries, IngestionRate, IngestionErrors, QueryErrors
aws cloudwatch get-metric-statistics --namespace AWS/Prometheus \
  --metric-name ActiveSeries --dimensions Name=WorkspaceId,Value=<workspace-id> \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 --statistics Maximum

# Check ingestion errors
aws cloudwatch get-metric-statistics --namespace AWS/Prometheus \
  --metric-name IngestionErrors --dimensions Name=WorkspaceId,Value=<workspace-id> \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 --statistics Sum
```

Triage returns:
- Workspace state, alias, creation timestamp
- Active series count and ingestion rate
- Ingestion and query error counts
- Workspace endpoint URLs (remote write and query)

If the workspace status is not ACTIVE, that IS the root cause domain. Don't chase ingestion or query symptoms.

### Step 2 — Domain deep dive (only if needed)

```
# Check remote write endpoint connectivity
curl -s -o /dev/null -w "%{http_code}" --aws-sigv4 "aws:amz:us-east-1:aps" \
  --user "$AWS_ACCESS_KEY_ID:$AWS_SECRET_ACCESS_KEY" \
  https://aps-workspaces.<region>.amazonaws.com/workspaces/<workspace-id>/api/v1/remote_write

# Check query endpoint
curl -s --aws-sigv4 "aws:amz:<region>:aps" \
  --user "$AWS_ACCESS_KEY_ID:$AWS_SECRET_ACCESS_KEY" \
  "https://aps-workspaces.<region>.amazonaws.com/workspaces/<workspace-id>/api/v1/query?query=up"

# List alert manager configuration
aws amp describe-alert-manager-definition --workspace-id <workspace-id>

# List ruler configuration
aws amp describe-rule-groups-namespace --workspace-id <workspace-id> --name <namespace>
aws amp list-rule-groups-namespaces --workspace-id <workspace-id>

# CloudTrail for AMP API events
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=aps.amazonaws.com --max-results 20
```

Read `references/amp-guardrails.md` before concluding on any AMP issue.

### Step 3 — Detailed investigation (low-confidence cases only)

```
# CloudTrail for detailed API call history
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=aps.amazonaws.com \
  --start-time $(date -u -d '24 hours ago' +%Y-%m-%dT%H:%M:%S) --max-results 50

# VPC endpoint check (if using private connectivity)
aws ec2 describe-vpc-endpoints --filters Name=service-name,Values=com.amazonaws.<region>.aps-workspaces

# IAM role/policy verification for SigV4
aws iam simulate-principal-policy --policy-source-arn <role-arn> \
  --action-names aps:RemoteWrite aps:QueryMetrics aps:GetSeries aps:GetLabels

# Check ADOT collector logs (if running on EKS)
kubectl logs -n opentelemetry -l app=adot-collector --tail=100

# Check Prometheus server remote write status (if using Prometheus)
kubectl logs -n monitoring -l app=prometheus --tail=100 | grep -i "remote"
```

## Tool quick reference

| Tool / Command | When to use |
|----------------|-------------|
| `aws amp list-workspaces` | List all workspaces, check status and aliases |
| `aws amp describe-workspace` | Workspace details, endpoints, status, tags |
| `aws amp describe-alert-manager-definition` | Alert manager YAML configuration |
| `aws amp list-rule-groups-namespaces` | List all ruler namespaces |
| `aws amp describe-rule-groups-namespace` | Ruler rules YAML for a namespace |
| `aws cloudwatch get-metric-statistics` | AMP metrics: ActiveSeries, IngestionRate, IngestionErrors |
| `aws cloudtrail lookup-events` | AMP API call history and errors |
| `aws iam simulate-principal-policy` | Verify SigV4 IAM permissions |
| `aws ec2 describe-vpc-endpoints` | VPC endpoint status for private connectivity |
| `curl --aws-sigv4` | Test remote write and query endpoints directly |
| `promtool check rules` | Validate Prometheus rule files locally |
| `kubectl logs` | ADOT collector or Prometheus server logs on EKS |

## Gotchas: AMP

These are the mistakes commonly made during AMP troubleshooting.

- AMP is a managed Prometheus-compatible service — it is NOT self-hosted Prometheus. There is no Prometheus server to SSH into, no local TSDB, no Prometheus UI. You interact via remote write (ingestion) and query (PromQL) API endpoints only.
- Remote write to AMP requires SigV4 authentication — basic auth, bearer tokens, and other Prometheus-native auth methods do NOT work. Every request must be signed with AWS credentials using the SigV4 protocol.
- Each workspace has ingestion rate limits and active series limits. Exceeding these causes HTTP 429 (throttling) responses on the remote write endpoint. Check CloudWatch `IngestionRate` and `ActiveSeries` metrics.
- The query endpoint is separate from the remote write endpoint. Remote write URL ends in `/api/v1/remote_write`. Query URL uses `/api/v1/query` and `/api/v1/query_range`. Mixing them up causes 404 or method-not-allowed errors.
- AMP does NOT run a Prometheus server — there is no local storage, no Prometheus UI, no targets page, no service discovery configuration. AMP is a remote storage backend only. Use Grafana or the query API for visualization.
- Alerting in AMP uses the ruler service with SNS integration for notifications. It does NOT use a standalone Alertmanager with email/Slack/PagerDuty receivers directly — you route through SNS topics which then fan out to your notification channels.
- Metrics retention in AMP is 150 days and is NOT configurable. You cannot extend or reduce it. Plan data lifecycle accordingly.
- HA collector setups (multiple Prometheus servers or ADOT collectors scraping the same targets) send duplicate samples. AMP deduplicates based on labels, but you must ensure consistent external labels across replicas to enable proper deduplication.
- ADOT Collector and Prometheus server are both valid remote write clients, but they have different configuration formats. ADOT uses `exporters.prometheusremotewrite` in its YAML config. Prometheus uses `remote_write` in `prometheus.yml`. Don't mix the config syntax.
- VPC endpoints are available for private connectivity to AMP (`com.amazonaws.<region>.aps-workspaces`). Without a VPC endpoint, traffic goes over the public internet even from within AWS. This matters for security-sensitive workloads.
- Workspace deletion is permanent — there is no recovery, no recycle bin, no snapshot. All metrics data is lost immediately. Always confirm the workspace ID before deleting.
- Label and series cardinality is the #1 performance issue in AMP. High-cardinality labels (like request IDs, pod UIDs, or timestamps in label values) explode the active series count, leading to throttling, slow queries, and increased costs.

## Anti-hallucination rules

1. Always cite specific AWS CLI output, CloudWatch metrics, CloudTrail events, or collector logs as evidence.
2. Never suggest accessing Prometheus server internals (TSDB, WAL, config reload endpoint) — AMP is fully managed with no server-side access.
3. Never recommend basic auth, bearer tokens, or API keys for AMP authentication — only SigV4 signing is supported.
4. Never claim metrics retention is configurable — it is fixed at 150 days.
5. Never suggest using Prometheus UI, targets page, or service discovery configuration — AMP has no UI; use Grafana or the query API.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 26 runbooks

Runbooks are organized by failure domain. Use the appropriate runbook based on the symptom category.

| Category | IDs | Covers |
|----------|-----|--------|
| A — Workspace | A1–A3 | Workspace creation failures, configuration issues, deletion/recovery |
| B — Ingestion | B1–B4 | Remote write failures, SigV4 auth errors, rate limiting, high cardinality |
| C — Query | C1–C3 | Query timeouts, PromQL errors, query performance |
| D — Alerting | D1–D3 | Rule evaluation failures, SNS integration, alert manager config |
| E — Collectors | E1–E3 | ADOT collector issues, Prometheus server remote write, EKS add-on |
| F — Security | F1–F2 | IAM/SigV4 configuration, VPC endpoint access |
| G — Integration | G1–G2 | Grafana data source, third-party tools |
| Z — Catch-All | Z1 | General AMP troubleshooting |
