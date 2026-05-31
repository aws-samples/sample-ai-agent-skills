---
title: "B1 — Remote Write Failures"
description: "Diagnose and resolve AMP remote write ingestion failures"
status: active
severity: HIGH
triggers:
  - "remote write failed"
  - "remote write error"
  - "ingestion failed"
  - "remote_write"
  - "4xx"
  - "5xx"
  - "metrics not appearing"
owner: devops-agent
objective: "Identify and fix remote write failures to restore metrics ingestion into AMP"
context: "Remote write is the only way to ingest metrics into AMP. Collectors (ADOT, Prometheus, Grafana Agent) send samples via HTTP POST to the remote write endpoint with SigV4 authentication. Failures can be caused by auth errors, endpoint misconfiguration, network issues, or rate limiting."
---

## Phase 1 — Triage

MUST:
- Check CloudWatch ingestion error metrics: `aws cloudwatch get-metric-statistics --namespace AWS/Prometheus --metric-name IngestionErrors --dimensions Name=WorkspaceId,Value=<workspace-id> --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) --end-time $(date -u +%Y-%m-%dT%H:%M:%S) --period 300 --statistics Sum`
- Verify workspace is ACTIVE: `aws amp describe-workspace --workspace-id <workspace-id> --query 'workspace.status.statusCode'`
- Test remote write endpoint directly: `curl -s -o /dev/null -w "%{http_code}" --aws-sigv4 "aws:amz:<region>:aps" --user "$AWS_ACCESS_KEY_ID:$AWS_SECRET_ACCESS_KEY" -X POST "https://aps-workspaces.<region>.amazonaws.com/workspaces/<workspace-id>/api/v1/remote_write"`
- Check collector logs for remote write errors (ADOT or Prometheus)

SHOULD:
- Verify the remote write URL format: must end with `/api/v1/remote_write`
- Check HTTP response codes in collector logs (403=auth, 404=wrong URL, 429=throttled, 5xx=service error)
- Verify SigV4 credentials are valid and not expired
- Check network connectivity from collector to AMP endpoint

MAY:
- Check CloudTrail for RemoteWrite API errors
- Verify DNS resolution of `aps-workspaces.<region>.amazonaws.com`
- Check if a VPC endpoint is configured and healthy

## Phase 2 — Remediate

MUST:
- If 403 Forbidden: fix SigV4 configuration and IAM permissions (see B2-sigv4-auth-errors)
- If 404 Not Found: correct the remote write URL — verify workspace ID and path
- If 429 Too Many Requests: reduce ingestion rate or request quota increase (see B3-rate-limiting)
- If 5xx Server Error: retry with exponential backoff; check AWS Health Dashboard for service issues

SHOULD:
- Ensure collector has retry logic with exponential backoff for transient failures
- Configure appropriate `queue_config` in Prometheus remote write for buffering
- Monitor `prometheus_remote_storage_failed_samples_total` metric in the collector

MAY:
- Set up CloudWatch alarms on IngestionErrors metric
- Implement a dead-letter mechanism for failed samples
- Configure multiple remote write endpoints for redundancy

## Common Issues

- symptoms: "Collector logs show 'remote write returned HTTP 403'"
  diagnosis: "SigV4 authentication failure — invalid credentials or missing IAM permissions."
  resolution: "Verify IAM role has aps:RemoteWrite permission. Check SigV4 config in collector. See B2-sigv4-auth-errors."

- symptoms: "Collector logs show 'remote write returned HTTP 404'"
  diagnosis: "Incorrect remote write URL — wrong workspace ID or missing /api/v1/remote_write path."
  resolution: "Verify URL: https://aps-workspaces.<region>.amazonaws.com/workspaces/<workspace-id>/api/v1/remote_write"

- symptoms: "Metrics not appearing in queries but no errors in collector logs"
  diagnosis: "Collector may be sending to wrong workspace, or query is using wrong time range."
  resolution: "Verify workspace ID in collector config matches query target. Check query time range covers ingestion time."

- symptoms: "Intermittent remote write failures with connection timeouts"
  diagnosis: "Network connectivity issue between collector and AMP endpoint."
  resolution: "Check security groups, NACLs, NAT gateway, or VPC endpoint configuration. Verify DNS resolution."

## Safety Ratings

```
safety_ratings:
  - "cloudwatch get-metric-statistics (IngestionErrors): GREEN — read-only metric check"
  - "describe-workspace: GREEN — read-only status check"
  - "test remote write endpoint: GREEN — connectivity test"
  - "fix collector remote write URL: YELLOW — changes collector configuration"
  - "modify collector queue_config: YELLOW — changes ingestion behavior"
```

## Escalation Conditions

- Remote write failures causing data loss in production monitoring
- HTTP 5xx errors from AMP endpoint indicating service-side issues
- All collectors failing simultaneously, suggesting a workspace or network issue
- Rate limiting (429) requires urgent quota increase for production workload

## Data Sensitivity

- `aws cloudwatch get-metric-statistics`: LOW — aggregated error counts
- Collector logs: MEDIUM — may contain endpoint URLs and error details
- Remote write endpoint URL: LOW — contains workspace ID but not credentials

## Prohibited Actions

- NEVER suggest deleting a workspace with active collectors writing to it
- NEVER suggest basic auth (only SigV4)
- NEVER disable retry logic in collectors to work around ingestion errors
- NEVER point multiple collectors at the same scrape targets without deduplication

## Phase 3 — Rollback

1. If collector configuration was changed incorrectly, restore the previous remote_write configuration
2. If queue_config was modified, restore original settings to prevent data loss
3. If the remote write URL was changed to a wrong workspace, restore the correct workspace ID
4. Verify metrics are flowing by checking CloudWatch IngestionRate metric
5. Confirm no data gap exists by querying recent metrics in the workspace

## Output Format

```yaml
root_cause: "Remote write failure — <specific_cause>"
evidence:
  - type: cloudwatch_metric
    content: "<IngestionErrors metric data>"
  - type: collector_logs
    content: "<relevant error messages from collector>"
severity: HIGH
mitigation:
  immediate: "Fix the identified auth, URL, or network issue"
  long_term: "Implement monitoring on IngestionErrors, configure retry logic in collector"
```

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
