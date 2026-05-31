---
title: "A2 — Instrumentation Errors"
description: "Diagnose ADOT auto-instrumentation and OpenTelemetry errors"
status: active
severity: HIGH
triggers: ["instrumentation error", "ADOT agent error", "OpenTelemetry error", "auto-instrumentation failed"]
owner: devops-agent
objective: "Resolve ADOT auto-instrumentation and OpenTelemetry configuration errors"
context: "Auto-instrumentation uses ADOT agents to inject OpenTelemetry instrumentation into applications at runtime."
---

## Phase 1 — Triage

MUST:
- Verify ADOT agent is running: check pod/process status
- Check ADOT agent logs for errors
- Verify the application language is supported (Java, Python, .NET, Node.js)

SHOULD:
- Check ADOT agent version compatibility with Application Signals
- Verify environment variables: OTEL_SERVICE_NAME, OTEL_EXPORTER_OTLP_ENDPOINT
- Check if the application framework is compatible with auto-instrumentation

MAY:
- Review OpenTelemetry SDK configuration if using manual instrumentation
- Check for conflicting instrumentation libraries

## Phase 2 — Remediate

MUST:
- Install the correct ADOT agent version for the application language
- Set required environment variables (OTEL_SERVICE_NAME, OTEL_RESOURCE_ATTRIBUTES)
- Ensure ADOT agent can reach the CloudWatch agent OTLP endpoint

SHOULD:
- Use the latest stable ADOT agent version
- Test instrumentation on a non-production instance first

MAY:
- Configure sampling to reduce overhead for high-throughput services
- Add custom spans for business-critical operations

## Common Issues

- symptoms: "Application crashes or fails to start after adding ADOT instrumentation"
  diagnosis: "ADOT agent version is incompatible with the application framework or JVM version."
  resolution: "Check ADOT compatibility matrix. Use a compatible agent version. For Java, ensure JVM version >= 8."

- symptoms: "Traces are generated but metrics are missing in Application Signals"
  diagnosis: "ADOT agent is configured for tracing only, not metrics."
  resolution: "Ensure ADOT agent configuration includes metric export to CloudWatch. Verify CloudWatch agent has Application Signals enabled."

## Safety Ratings

```
safety_ratings:
  - "Check ADOT agent logs: GREEN — read-only log review"
  - "Verify environment variables: GREEN — read-only configuration check"
  - "Update ADOT agent version: YELLOW — requires application restart"
  - "Modify instrumentation configuration: YELLOW — affects application behavior"
```

## Escalation Conditions

- Instrumentation causes application performance degradation exceeding 5%
- Auto-instrumentation is incompatible with the application framework
- ADOT agent crashes repeatedly after configuration fixes
- Custom instrumentation required for unsupported language or framework

## Data Sensitivity

- ADOT agent logs: MEDIUM — may contain application endpoint and configuration details
- Environment variables: MEDIUM — may contain endpoint URLs and configuration
- OpenTelemetry spans: MEDIUM — may contain request paths and parameters
- Application startup logs: MEDIUM — may reveal framework and dependency versions

## Prohibited Actions

- NEVER deploy untested ADOT agent versions to production
- NEVER disable all instrumentation to fix a single instrumentation error
- NEVER set OTEL_TRACES_SAMPLER to always_off in production without approval
- NEVER ignore instrumentation-related application crashes

## Phase 3 — Rollback

1. Remove ADOT agent from the application: remove annotations (EKS) or uninstall agent (EC2)
2. Restart the application without instrumentation
3. Verify application performance returns to baseline
4. Revert environment variable changes
5. Document the instrumentation error and agent version for future reference

## Output Format

```yaml
root_cause: "instrumentation_error — <specific_cause>"
evidence:
  - type: agent_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix agent configuration or version"
  long_term: "Standardize ADOT agent deployment"
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
