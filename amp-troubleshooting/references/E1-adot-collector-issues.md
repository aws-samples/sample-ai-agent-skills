---
title: "E1 — ADOT Collector Issues"
description: "Diagnose and resolve AWS Distro for OpenTelemetry (ADOT) Collector issues with AMP"
status: active
severity: HIGH
triggers:
  - "ADOT"
  - "adot-collector"
  - "OpenTelemetry"
  - "otel"
  - "collector not sending"
  - "prometheusremotewrite"
owner: devops-agent
objective: "Identify and fix ADOT Collector configuration and connectivity issues with AMP"
context: "ADOT Collector is the AWS-supported OpenTelemetry collector for sending metrics to AMP. It uses the prometheusremotewrite exporter with sigv4auth extension for authentication. Configuration is in OpenTelemetry YAML format, NOT Prometheus format."
---

## Phase 1 — Triage

MUST:
- Check ADOT collector pod/process status:
  - On EKS: `kubectl get pods -n <namespace> -l app=adot-collector`
  - On EC2: `systemctl status adot-collector`
- Check collector logs for errors:
  - On EKS: `kubectl logs -n <namespace> -l app=adot-collector --tail=100`
  - On EC2: `journalctl -u adot-collector --since "1 hour ago"`
- Verify the collector configuration has correct AMP endpoint and SigV4 settings
- Check if the collector is scraping targets: look for `scrape` or `receiver` logs

SHOULD:
- Verify the prometheusremotewrite exporter configuration:
  ```yaml
  exporters:
    prometheusremotewrite:
      endpoint: "https://aps-workspaces.<region>.amazonaws.com/workspaces/<workspace-id>/api/v1/remote_write"
      auth:
        authenticator: sigv4auth
  ```
- Verify the sigv4auth extension:
  ```yaml
  extensions:
    sigv4auth:
      region: "<region>"
      service: "aps"
  ```
- Check IAM credentials available to the collector (IRSA, instance profile, env vars)
- Verify the collector's service pipeline includes the exporter

MAY:
- Enable debug logging in the collector for more detail
- Check collector self-metrics (if exposed) for export errors
- Verify network connectivity from collector to AMP endpoint

## Phase 2 — Remediate

MUST:
- If endpoint wrong: fix the prometheusremotewrite endpoint URL
- If SigV4 not configured: add sigv4auth extension and reference it in the exporter
- If IAM credentials missing: configure IRSA (EKS), instance profile (EC2), or environment variables
- If pipeline misconfigured: ensure the exporter is included in the service pipelines section

SHOULD:
- Use the latest ADOT Collector version for bug fixes and AMP compatibility
- Configure retry and queue settings in the exporter for resilience
- Set up health check endpoint for collector monitoring

MAY:
- Configure multiple exporters for sending to both AMP and other backends
- Implement collector horizontal scaling for high-volume environments
- Set up collector self-monitoring metrics

## Common Issues

- symptoms: "ADOT collector logs show 'Permanent error: remote write returned HTTP 403'"
  diagnosis: "SigV4 authentication failure — missing or invalid credentials."
  resolution: "Verify sigv4auth extension config. Check IAM role has aps:RemoteWrite. Verify IRSA setup on EKS."

- symptoms: "Collector running but no metrics in AMP"
  diagnosis: "Exporter not included in service pipeline, or scraping no targets."
  resolution: "Check service.pipelines in config includes the prometheusremotewrite exporter. Verify receiver is scraping targets."

- symptoms: "Collector CrashLoopBackOff on EKS"
  diagnosis: "Configuration error causing collector to fail on startup."
  resolution: "Check logs for config parsing errors. Validate YAML syntax. Ensure all referenced extensions/exporters are defined."

- symptoms: "Collector logs show 'context deadline exceeded'"
  diagnosis: "Network timeout connecting to AMP endpoint."
  resolution: "Check security groups, NAT gateway, VPC endpoint. Verify DNS resolution of aps-workspaces endpoint."

## Safety Ratings

```
safety_ratings:
  - "check collector pod/process status: GREEN — read-only status check"
  - "check collector logs: GREEN — read-only log review"
  - "fix collector configuration: YELLOW — changes data collection behavior"
  - "update ADOT collector version: YELLOW — software update on infrastructure"
```

## Escalation Conditions

- ADOT collector CrashLoopBackOff affecting production metrics collection
- Collector configuration changes needed across multiple clusters
- SigV4 authentication failures requiring IRSA or Pod Identity changes
- Collector network timeouts indicating infrastructure connectivity issues

## Data Sensitivity

- Collector configuration: MEDIUM — reveals AMP endpoint, scrape targets, and auth settings
- Collector logs: MEDIUM — may contain endpoint URLs and error details
- IRSA configuration: MEDIUM — reveals service account trust relationships

## Prohibited Actions

- NEVER suggest deleting a workspace with active collectors writing to it
- NEVER suggest basic auth (only SigV4)
- NEVER deploy untested collector configurations to production
- NEVER mix OpenTelemetry YAML format with Prometheus YAML format

## Phase 3 — Rollback

1. If collector configuration was changed, restore previous ConfigMap or configuration file
2. If ADOT version was updated, roll back to previous version via deployment/daemonset update
3. If IRSA was modified, restore previous service account annotation and trust policy
4. Verify collector pods are running and metrics are flowing after rollback
5. Check CloudWatch IngestionRate to confirm data ingestion resumed

## Output Format

```yaml
root_cause: "ADOT collector — <specific_cause>"
evidence:
  - type: collector_logs
    content: "<relevant error messages>"
  - type: collector_config
    content: "<configuration analysis>"
severity: HIGH
mitigation:
  immediate: "Fix collector configuration or connectivity issue"
  long_term: "Implement collector monitoring, use IaC for configuration management"
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
