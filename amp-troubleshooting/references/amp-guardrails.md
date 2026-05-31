# Amazon Managed Service for Prometheus Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any AMP issue.

## Guardrail 1: SigV4 Is the Only Authentication Method
AMP requires AWS Signature Version 4 (SigV4) for all API requests — both remote write and query. Basic auth, bearer tokens, API keys, and other Prometheus-native auth methods are NOT supported. Every HTTP request must include SigV4 headers. Collectors (ADOT, Prometheus) must be configured with SigV4 signing using valid IAM credentials. If you see 403 Forbidden, check IAM permissions AND SigV4 configuration — not auth tokens.

## Guardrail 2: Ingestion Rate Limits Are Per-Workspace
Each AMP workspace has ingestion rate limits measured in samples per second. Exceeding the limit returns HTTP 429 (Too Many Requests) on the remote write endpoint. Check CloudWatch `IngestionRate` metric to see current throughput. If consistently hitting limits, request a quota increase via AWS Service Quotas or reduce scrape frequency/targets. Do NOT assume the limit is infinite.

## Guardrail 3: Active Series Limits Are Per-Workspace
Each workspace has a maximum number of active time series. An active series is any series that received a sample in the last 15 minutes. Exceeding this limit causes ingestion rejections. Check CloudWatch `ActiveSeries` metric. The #1 cause of hitting this limit is high-cardinality labels (pod UIDs, request IDs, timestamps in labels). Reducing cardinality is the fix — not increasing the limit.

## Guardrail 4: Retention Is Fixed at 150 Days
AMP retains metrics for 150 days. This is NOT configurable — you cannot extend it, reduce it, or set per-metric retention policies. Data older than 150 days is automatically deleted. If you need longer retention, export data to S3 using a separate pipeline. Do NOT suggest changing retention settings — the option does not exist.

## Guardrail 5: No Prometheus UI — Query via API or Grafana
AMP has no built-in UI. There is no Prometheus expression browser, no targets page, no service discovery view, no TSDB status page. To query metrics, use the Prometheus-compatible query API (`/api/v1/query`, `/api/v1/query_range`) with SigV4 signing, or configure Amazon Managed Grafana or self-hosted Grafana with the AMP data source. Do NOT tell users to "open the Prometheus UI."

## Guardrail 6: Alerting Uses Ruler + SNS (Not Standalone AlertManager)
AMP alerting works through the ruler service that evaluates recording and alerting rules. Alert notifications are sent via Amazon SNS — not directly to Slack, PagerDuty, or email. The alert manager definition in AMP configures SNS as the receiver. From SNS, you fan out to your preferred notification channels (Lambda, email, Slack via Chatbot, etc.). Do NOT suggest configuring email/Slack receivers directly in the AMP alert manager.

## Guardrail 7: Workspaces Are Region-Bound
Each AMP workspace exists in a single AWS region. There is no cross-region replication, no global workspace, and no multi-region failover built in. If you need metrics in multiple regions, create separate workspaces and configure collectors in each region. For cross-region querying, use Grafana with multiple AMP data sources.

## Guardrail 8: Remote Write and Query Endpoints Are Different
The remote write endpoint format is: `https://aps-workspaces.<region>.amazonaws.com/workspaces/<workspace-id>/api/v1/remote_write`. The query endpoint format is: `https://aps-workspaces.<region>.amazonaws.com/workspaces/<workspace-id>/api/v1/query`. These are different paths on the same host. Using the wrong endpoint returns 404 or 405 errors. Verify the full URL including the path suffix.

## Guardrail 9: ADOT and Prometheus Have Different Config Formats
AWS Distro for OpenTelemetry (ADOT) Collector uses OpenTelemetry configuration YAML with `exporters.prometheusremotewrite` and `extensions.sigv4auth`. Prometheus server uses `prometheus.yml` with `remote_write` section and `sigv4` sub-config. These formats are NOT interchangeable. Mixing them causes silent config parsing failures. Always verify which collector is in use before suggesting configuration changes.

## Guardrail 10: Label Cardinality Is the #1 Performance Problem
High-cardinality labels create an explosion of unique time series. Common offenders: pod UIDs (`pod` label with full UID), request/trace IDs, user IDs, timestamps in label values, and auto-generated names. Each unique label combination creates a new series. A single metric with a high-cardinality label can generate millions of series. Always check `count({__name__="<metric>"})` to assess cardinality before investigating performance issues.

## Guardrail 11: Workspace Deletion Is Permanent
Deleting an AMP workspace permanently destroys all metrics data, alert manager definitions, and ruler configurations. There is no soft delete, no recycle bin, no snapshot, and no recovery mechanism. Always double-check the workspace ID before deletion. Consider exporting critical rule configurations before deleting a workspace.

## Guardrail 12: VPC Endpoint Provides Private Connectivity
AMP supports VPC endpoints (`com.amazonaws.<region>.aps-workspaces`) for private connectivity. Without a VPC endpoint, all traffic from VPC-based collectors traverses the public internet (via NAT gateway or internet gateway). For security-sensitive environments, configure the VPC endpoint and ensure the security group allows HTTPS (443) from collector subnets. The VPC endpoint does NOT change the endpoint URL — DNS resolution handles the routing.
