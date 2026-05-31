# Amazon Lookout for Metrics Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Lookout for Metrics issue.

## Guardrail 1: Mandatory Learning Period
Detectors require a cold-start learning period of 1–2 weeks after activation before producing reliable anomaly results. During this period, the detector is building a baseline model. Do not diagnose "no anomalies detected" as a failure during the learning window.

## Guardrail 2: Data Quality Is the Primary Diagnostic
Always check data quality metrics (`get-data-quality-metrics`) before investigating detection logic. Most anomaly detection failures trace back to incomplete, malformed, or missing data — not model issues.

## Guardrail 3: S3 Path Templates Must Match Exactly
S3 data sources require date-partitioned folder structures with a specific path template (e.g., `s3://bucket/prefix/{{yyyyMMdd}}/{{HHmm}}/`). Even minor deviations — wrong date format, missing trailing slash, incorrect partition granularity — cause silent ingestion failures with no error in the detector status.

## Guardrail 4: Timestamp Format Is the Top Configuration Error
The timestamp format in the metric set must exactly match the format in the source data. Common mistakes: using `yyyy-MM-dd` when data has `yyyyMMdd`, or omitting timezone information. Mismatched timestamps cause the detector to see zero data points.

## Guardrail 5: Deactivation Resets the Learning Period
Deactivating a detector and reactivating it resets the learning period entirely. The model must rebuild from scratch. Do not casually suggest deactivate/reactivate cycles — each one costs 1–2 weeks of learning time.

## Guardrail 6: Sensitivity Threshold Is Not Binary
The sensitivity threshold (1–100) is a continuous scale. Lower values (e.g., 25) detect only severe anomalies with high confidence. Higher values (e.g., 75) detect subtle anomalies but increase false positives. There is no universally correct value — it depends on the use case.

## Guardrail 7: Metric Sets Define the Detection Scope
Each metric set defines measures (numeric values to monitor), dimensions (categorical groupings), and a timestamp column. A detector can have multiple metric sets. Misconfigured metric sets are the second most common root cause after timestamp issues.

## Guardrail 8: CloudWatch Data Sources Require Exact Matches
CloudWatch data sources require the metric namespace, metric name, and dimension name/value pairs to match exactly. Partial matches, wildcards, or regex are not supported. Verify metric existence with `aws cloudwatch list-metrics` before configuring.

## Guardrail 9: Lambda Alert Permissions Are Separate
Lambda function alerts require a resource-based policy on the Lambda function granting `lambda:InvokeFunction` to the Lookout for Metrics service principal. IAM role permissions alone are insufficient — the function policy must explicitly allow invocation.

## Guardrail 10: Anomaly Groups Are Not Individual Anomalies
Lookout for Metrics groups related anomalies into anomaly groups based on correlated metrics and dimensions. A single anomaly group may contain multiple affected metrics. Do not treat anomaly group count as individual anomaly count.

## Guardrail 11: Redshift and RDS Require VPC Connectivity
Redshift and RDS data sources require the Lookout for Metrics service to reach the database endpoint. This requires proper VPC configuration, security group rules, and subnet routing. The service does not use VPC endpoints — it connects via the public or private endpoint.

## Guardrail 12: Back-Test Mode Has Different Behavior
Back-test mode uses historical data to evaluate detector performance before activation. Results from back-test mode may differ from live detection because the model has access to the full historical dataset. Do not use back-test results as the sole indicator of live performance.
