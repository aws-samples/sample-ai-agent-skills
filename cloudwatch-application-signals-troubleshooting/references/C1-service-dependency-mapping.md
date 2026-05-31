---
title: "C1 — Service Dependency Mapping"
description: "Diagnose CloudWatch Application Signals service dependency mapping issues"
status: active
severity: MEDIUM
triggers: ["missing dependency", "dependency map incomplete", "service map wrong", "dependency not showing"]
owner: devops-agent
objective: "Resolve service dependency mapping and topology issues"
context: "Dependency maps are built from distributed trace data. Incomplete tracing results in incomplete maps."
---

## Phase 1 — Triage

MUST:
- List service dependencies: `aws application-signals list-service-dependencies --start-time $(date -d '-1 hour' -u +%s) --end-time $(date -u +%s) --key-attributes Name=<service-name>,Type=<type>,Environment=<env>`
- Verify X-Ray tracing is enabled for all services in the dependency chain
- Check trace context propagation between services

SHOULD:
- Review trace samples for missing spans: `aws xray get-trace-summaries --start-time $(date -d '-1 hour' -u +%Y-%m-%dT%H:%M:%S) --end-time $(date -u +%Y-%m-%dT%H:%M:%S)`
- Verify all services in the chain are instrumented with ADOT

MAY:
- Check for trace context propagation headers (traceparent, X-Amzn-Trace-Id)
- Review service-to-service communication patterns

## Phase 2 — Remediate

MUST:
- Instrument all services in the dependency chain with ADOT
- Enable X-Ray tracing for all services
- Ensure trace context headers are propagated between services

SHOULD:
- Verify HTTP clients and message queue consumers propagate trace context
- Check that load balancers and API gateways forward trace headers

MAY:
- Add custom spans for external service calls
- Configure trace sampling to capture sufficient dependency data

## Common Issues

- symptoms: "Dependency map shows service A but not its downstream service B"
  diagnosis: "Service B is not instrumented or trace context is not propagated."
  resolution: "Instrument service B with ADOT. Ensure HTTP client in service A propagates trace context headers."

- symptoms: "Dependencies appear intermittently in the map"
  diagnosis: "Trace sampling rate is too low to capture all dependency paths."
  resolution: "Increase sampling rate or use reservoir sampling to ensure minimum trace capture per dependency."

## Safety Ratings

```
safety_ratings:
  - "list-service-dependencies: GREEN — read-only dependency listing"
  - "xray get-trace-summaries: GREEN — read-only trace data"
  - "Instrumentation changes: YELLOW — requires application restart"
  - "Sampling configuration changes: YELLOW — affects trace volume"
```

## Escalation Conditions

- Critical dependency is missing from the map despite both services being instrumented
- Trace context propagation fails through a middleware or proxy
- Dependency mapping required for services using unsupported communication protocols
- Cross-region dependencies are not captured

## Data Sensitivity

- `aws application-signals list-service-dependencies`: LOW — returns dependency metadata
- `aws xray get-trace-summaries`: MEDIUM — reveals service communication patterns
- Trace data: MEDIUM — may contain request paths, headers, and timing
- Service topology: MEDIUM — reveals architecture and communication patterns

## Prohibited Actions

- NEVER assume a missing dependency means services are not communicating
- NEVER disable trace sampling to capture all traces in production (causes overhead)
- NEVER modify trace context headers manually — use SDK propagation
- NEVER ignore missing dependencies in critical service paths

## Phase 3 — Rollback

1. If instrumentation changes caused issues, revert ADOT configuration
2. Restore original sampling configuration if it was modified
3. Verify application performance after reverting changes
4. Confirm existing dependency mappings are still accurate
5. Document which dependencies are missing and why

## Output Format

```yaml
root_cause: "dependency_mapping — <specific_cause>"
evidence:
  - type: dependency_error
    content: "<missing dependency details>"
severity: MEDIUM
mitigation:
  immediate: "Fix instrumentation or trace propagation"
  long_term: "Ensure all services in critical paths are instrumented"
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
