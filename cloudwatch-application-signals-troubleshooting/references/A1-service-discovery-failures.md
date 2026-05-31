---
title: "A1 — Service Discovery Failures"
description: "Diagnose CloudWatch Application Signals service discovery failures"
status: active
severity: HIGH
triggers: ["service not discovered", "service not showing", "missing service", "no services found"]
owner: devops-agent
objective: "Resolve service discovery issues in Application Signals"
context: "Service discovery depends on ADOT auto-instrumentation injecting correct OpenTelemetry resource attributes."
---

## Phase 1 — Triage

MUST:
- List discovered services: `aws application-signals list-services --start-time $(date -d '-1 hour' -u +%s) --end-time $(date -u +%s)`
- Verify CloudWatch agent is running on the host/pod
- Verify ADOT auto-instrumentation agent is deployed and injecting attributes

SHOULD:
- Check OpenTelemetry resource attributes (service.name, deployment.environment, telemetry.sdk.language)
- Verify metrics exist in AWS/ApplicationSignals namespace: `aws cloudwatch list-metrics --namespace 'AWS/ApplicationSignals'`

MAY:
- Check ADOT agent logs for instrumentation errors
- Verify the application is generating traffic (idle services may not appear)

## Phase 2 — Remediate

MUST:
- Deploy ADOT auto-instrumentation agent if missing
- Configure correct resource attributes (service.name, deployment.environment)
- Ensure CloudWatch agent configuration includes Application Signals section

SHOULD:
- Restart application pods/processes after instrumentation changes
- Verify metrics appear in CloudWatch within 5 minutes of fix

MAY:
- Use OpenTelemetry SDK for manual instrumentation if auto-instrumentation is not supported
- Configure custom service.name for disambiguation

## Common Issues

- symptoms: "Service does not appear in Application Signals console"
  diagnosis: "ADOT auto-instrumentation agent is not deployed or not injecting resource attributes."
  resolution: "Deploy ADOT agent. For EKS, annotate pods with instrumentation.opentelemetry.io/inject-<language>: 'true'. Restart pods."

- symptoms: "Service appears with generic name like 'unknown_service'"
  diagnosis: "service.name resource attribute is not set or defaults to unknown."
  resolution: "Set OTEL_SERVICE_NAME environment variable or configure service.name in ADOT agent configuration."

## Safety Ratings

```
safety_ratings:
  - "application-signals list-services: GREEN — read-only service enumeration"
  - "cloudwatch list-metrics: GREEN — read-only metric listing"
  - "Pod annotation for instrumentation: YELLOW — triggers pod restart"
  - "ADOT agent deployment: YELLOW — adds sidecar/daemonset to cluster"
```

## Escalation Conditions

- Service discovery fails after confirming both agents are running and configured
- Application language is not supported by ADOT auto-instrumentation
- Instrumentation causes application performance degradation
- Service appears intermittently, suggesting metric collection instability

## Data Sensitivity

- `aws application-signals list-services`: LOW — returns service names and metadata
- `aws cloudwatch list-metrics`: LOW — returns metric names and dimensions
- ADOT agent logs: MEDIUM — may contain application endpoint information
- OpenTelemetry resource attributes: LOW — contain service metadata

## Prohibited Actions

- NEVER disable instrumentation on production services without a rollback plan
- NEVER modify ADOT agent configuration on all pods simultaneously — use canary deployment
- NEVER assume a service is healthy because it does not appear in Application Signals
- NEVER set service.name to a value that conflicts with another service

## Phase 3 — Rollback

1. If instrumentation changes caused issues, remove ADOT annotations and restart pods
2. Revert CloudWatch agent configuration to pre-change state
3. If ADOT DaemonSet was deployed, scale it down: `kubectl scale daemonset adot-collector -n amazon-cloudwatch --replicas=0`
4. Verify application performance returns to baseline after removing instrumentation
5. Document the instrumentation configuration that was attempted

## Output Format

```yaml
root_cause: "service_discovery — <specific_cause>"
evidence:
  - type: discovery_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix instrumentation configuration"
  long_term: "Standardize instrumentation across services"
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
  - "NEVER suggest deleting alarms without review"
  - "NEVER reduce log retention below compliance requirements"
  - "NEVER disable detailed monitoring in production"
