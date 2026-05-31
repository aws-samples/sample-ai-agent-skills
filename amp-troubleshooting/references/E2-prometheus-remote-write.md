---
title: "E2 — Prometheus Server Remote Write"
description: "Diagnose and resolve Prometheus server remote write configuration for AMP"
status: active
severity: HIGH
triggers:
  - "prometheus remote write"
  - "prometheus.yml"
  - "remote_write"
  - "prometheus server"
  - "prometheus config"
owner: devops-agent
objective: "Identify and fix Prometheus server remote write configuration for AMP"
context: "Prometheus server can remote-write to AMP using the built-in remote_write configuration with SigV4 signing. The configuration is in prometheus.yml format, NOT OpenTelemetry format. SigV4 support was added in Prometheus 2.26+."
---

## Phase 1 — Triage

MUST:
- Check Prometheus server status and logs:
  - On EKS: `kubectl logs -n monitoring -l app=prometheus --tail=100 | grep -i "remote"`
  - On EC2: `journalctl -u prometheus --since "1 hour ago" | grep -i "remote"`
- Verify remote_write configuration in prometheus.yml
- Check Prometheus remote write metrics: `prometheus_remote_storage_failed_samples_total`, `prometheus_remote_storage_succeeded_samples_total`
- Verify Prometheus version supports SigV4 (2.26+)

SHOULD:
- Verify the remote_write configuration:
  ```yaml
  remote_write:
    - url: "https://aps-workspaces.<region>.amazonaws.com/workspaces/<workspace-id>/api/v1/remote_write"
      sigv4:
        region: "<region>"
      queue_config:
        max_samples_per_send: 1000
        max_shards: 200
        capacity: 2500
  ```
- Check if SigV4 credentials are available to the Prometheus process
- Verify queue_config settings are appropriate for the ingestion volume
- Check WAL (Write-Ahead Log) for backlog: `prometheus_remote_storage_pending_samples`

MAY:
- Check Prometheus TSDB status for local storage issues affecting remote write
- Verify no write_relabel_configs are dropping all metrics
- Check if external_labels are configured for HA deduplication

## Phase 2 — Remediate

MUST:
- If SigV4 not configured: add sigv4 block to remote_write section
- If wrong URL: fix the endpoint URL format
- If credentials missing: configure IAM credentials (IRSA on EKS, instance profile on EC2)
- If Prometheus version too old: upgrade to 2.26+ for SigV4 support

SHOULD:
- Configure queue_config for optimal throughput:
  ```yaml
  queue_config:
    max_samples_per_send: 1000
    max_shards: 200
    capacity: 2500
  ```
- Set external_labels for HA setups to enable deduplication
- Configure write_relabel_configs to filter unnecessary metrics before sending

MAY:
- Set up Prometheus Operator with remote write configuration via CRD
- Implement recording rules locally to reduce remote write volume
- Configure multiple remote_write endpoints for redundancy

## Common Issues

- symptoms: "Prometheus logs show 'non-recoverable error' for remote write"
  diagnosis: "Permanent failure — usually 403 (auth) or 404 (wrong URL)."
  resolution: "Check SigV4 config and IAM permissions. Verify endpoint URL."

- symptoms: "prometheus_remote_storage_pending_samples increasing"
  diagnosis: "Remote write is falling behind — samples are queuing up."
  resolution: "Increase max_shards and capacity in queue_config. Check for rate limiting (429s)."

- symptoms: "Remote write working but some metrics missing in AMP"
  diagnosis: "write_relabel_configs may be dropping metrics, or external_labels conflict."
  resolution: "Review write_relabel_configs. Check if metrics are being filtered out."

- symptoms: "'sigv4' is not a recognized remote_write config field"
  diagnosis: "Prometheus version is older than 2.26 — SigV4 not supported."
  resolution: "Upgrade Prometheus to 2.26 or later. Alternatively, use a SigV4 proxy."

## Safety Ratings

```
safety_ratings:
  - "check Prometheus logs: GREEN — read-only log review"
  - "check remote write metrics: GREEN — read-only metric check"
  - "fix remote_write configuration: YELLOW — changes data ingestion behavior"
  - "upgrade Prometheus version: YELLOW — software update on infrastructure"
  - "modify write_relabel_configs: YELLOW — changes which metrics are sent"
```

## Escalation Conditions

- Prometheus remote write failures causing data loss in production
- Prometheus version upgrade needed for SigV4 support
- Queue backlog growing, indicating sustained ingestion issues
- write_relabel_configs accidentally dropping critical metrics

## Data Sensitivity

- Prometheus configuration: MEDIUM — reveals scrape targets, remote write endpoints, and auth settings
- Prometheus logs: MEDIUM — may contain endpoint URLs and error details
- Remote write metrics: LOW — aggregated success/failure counts

## Prohibited Actions

- NEVER suggest deleting a workspace with active collectors writing to it
- NEVER suggest basic auth (only SigV4)
- NEVER use a SigV4 proxy in production without proper IAM role configuration
- NEVER modify write_relabel_configs without auditing which metrics are used in dashboards and alerts

## Phase 3 — Rollback

1. If remote_write configuration was changed, restore previous prometheus.yml
2. If Prometheus was upgraded and broke, roll back to previous version
3. If write_relabel_configs were modified, restore original filter rules
4. Restart Prometheus to apply configuration changes
5. Verify remote write metrics show successful sample delivery after rollback

## Output Format

```yaml
root_cause: "Prometheus remote write — <specific_cause>"
evidence:
  - type: prometheus_logs
    content: "<remote write error messages>"
  - type: prometheus_config
    content: "<remote_write configuration analysis>"
severity: HIGH
mitigation:
  immediate: "Fix remote write configuration or credentials"
  long_term: "Monitor remote write metrics, implement queue tuning, version control config"
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
