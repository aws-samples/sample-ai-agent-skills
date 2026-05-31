---
title: "G2 — Third-Party Tool Integration"
description: "Diagnose and resolve third-party tool integration issues with AMP"
status: active
severity: MEDIUM
triggers:
  - "third-party"
  - "integration"
  - "Datadog"
  - "Thanos"
  - "Cortex"
  - "Grafana Agent"
  - "Mimir"
  - "SigV4 proxy"
owner: devops-agent
objective: "Identify and fix integration issues between third-party tools and AMP"
context: "AMP exposes Prometheus-compatible APIs, allowing integration with tools that support Prometheus remote write or query. However, all requests must use SigV4 authentication. Tools that don't natively support SigV4 need a signing proxy. Common integrations include Grafana Agent, Thanos, Mimir, and custom applications."
---

## Phase 1 — Triage

MUST:
- Identify the third-party tool and its version
- Check if the tool supports SigV4 natively or needs a signing proxy
- Verify the tool's configuration for AMP endpoint URL and authentication
- Check tool logs for connection or authentication errors

SHOULD:
- Verify the tool uses the correct API paths (remote write vs query)
- Check if the tool's Prometheus compatibility version matches AMP's supported version
- Test connectivity from the tool to AMP endpoint
- Verify IAM credentials are available to the tool

MAY:
- Check if a SigV4 signing proxy is needed and properly configured
- Review the tool's documentation for AMP-specific integration guides
- Check for known compatibility issues between the tool version and AMP

## Phase 2 — Remediate

MUST:
- If tool lacks SigV4 support: deploy AWS SigV4 Proxy as a sidecar or standalone:
  ```
  # SigV4 proxy forwards requests with SigV4 signing
  aws-sigv4-proxy --name aps --region <region> --host aps-workspaces.<region>.amazonaws.com
  ```
- If endpoint URL wrong: update to correct AMP endpoint format
- If credentials missing: configure IAM credentials for the tool or proxy
- If API incompatibility: check AMP's supported Prometheus API version

SHOULD:
- Use the AWS SigV4 Proxy for tools without native SigV4 support
- Configure the proxy with least-privilege IAM role
- Test the integration end-to-end after configuration

MAY:
- Set up monitoring for the SigV4 proxy health
- Document the integration architecture for the team
- Implement automated testing for integration health

## Common Issues

- symptoms: "Tool returns 403 when writing to AMP"
  diagnosis: "Tool doesn't support SigV4 and is sending unsigned requests."
  resolution: "Deploy AWS SigV4 Proxy between the tool and AMP. Point the tool at the proxy."

- symptoms: "Grafana Agent remote write fails with connection refused"
  diagnosis: "Grafana Agent endpoint URL or SigV4 configuration is incorrect."
  resolution: "Verify Grafana Agent remote write config uses correct AMP URL and sigv4 block."

- symptoms: "Thanos sidecar cannot query AMP"
  diagnosis: "Thanos uses its own query protocol, not direct Prometheus API."
  resolution: "Use Thanos with a Prometheus instance that remote-writes to AMP. Or use Thanos query frontend pointing at AMP query endpoint with SigV4 proxy."

- symptoms: "Custom application gets 'SignatureDoesNotMatch' error"
  diagnosis: "Application's SigV4 signing implementation is incorrect."
  resolution: "Use AWS SDK for SigV4 signing. Verify region, service name (aps), and credential chain. Check system clock sync."

## Safety Ratings

```
safety_ratings:
  - "check tool logs: GREEN — read-only log review"
  - "verify tool configuration: GREEN — read-only config check"
  - "deploy SigV4 proxy: YELLOW — deploys new infrastructure component"
  - "configure tool endpoint: YELLOW — changes tool's data destination"
```

## Escalation Conditions

- Third-party tool integration failure affects production monitoring pipeline
- SigV4 proxy deployment requires infrastructure provisioning
- Tool version incompatibility with AMP's Prometheus API version
- Custom application SigV4 signing implementation needs debugging

## Data Sensitivity

- Tool configuration: MEDIUM — reveals integration architecture and endpoints
- Tool logs: MEDIUM — may contain error details and endpoint URLs
- SigV4 proxy configuration: MEDIUM — reveals IAM role and endpoint details

## Prohibited Actions

- NEVER suggest deleting a workspace with active collectors writing to it
- NEVER suggest basic auth (only SigV4)
- NEVER expose the SigV4 proxy to the public internet
- NEVER use the SigV4 proxy with overly permissive IAM roles

## Phase 3 — Rollback

1. If SigV4 proxy was deployed and caused issues, stop or remove the proxy
2. If tool configuration was changed, restore previous endpoint and auth settings
3. If IAM roles were created for the proxy, delete them if no longer needed
4. Restore the previous integration method if the new approach failed
5. Verify the tool can send/query data through the restored configuration

## Output Format

```yaml
root_cause: "Third-party integration — <specific_cause>"
evidence:
  - type: tool_logs
    content: "<error messages from the third-party tool>"
  - type: integration_config
    content: "<tool configuration for AMP>"
severity: MEDIUM
mitigation:
  immediate: "Fix tool configuration or deploy SigV4 proxy"
  long_term: "Document integration architecture, implement health monitoring"
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
