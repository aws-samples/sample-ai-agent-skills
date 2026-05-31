---
title: "G2 — App Runner X-Ray Tracing Issues"
description: "Diagnose X-Ray tracing configuration and missing trace problems"
status: active
severity: LOW
triggers:
  - "X-Ray"
  - "tracing"
  - "traces missing"
  - "observability"
  - "distributed tracing"
  - "trace not showing"
owner: devops-agent
objective: "Identify and fix X-Ray tracing configuration issues for App Runner services"
context: "App Runner supports X-Ray tracing through an observability configuration. When enabled, App Runner automatically runs the X-Ray daemon — the application only needs the X-Ray SDK or OpenTelemetry SDK. Common issues include missing observability configuration, application not instrumented, X-Ray daemon conflicts, and IAM permissions for trace submission."
---

## Phase 1 — Triage

MUST:
- Check observability configuration: `aws apprunner describe-service --service-arn <service-arn> --query 'Service.ObservabilityConfiguration'`
- If observability ARN exists, describe it: `aws apprunner describe-observability-configuration --observability-configuration-arn <obs-arn>`
- Check if the application is instrumented with X-Ray SDK or OpenTelemetry
- Review application logs for X-Ray SDK errors
- Check X-Ray console for traces: `aws xray get-trace-summaries --start-time <start> --end-time <end>`

SHOULD:
- Verify the observability configuration has tracing enabled (TraceConfiguration.Vendor = AWSXRAY)
- Check if the application is sending traces to the correct X-Ray daemon endpoint (localhost:2000)
- Verify the instance role has xray:PutTraceSegments and xray:PutTelemetryRecords permissions

MAY:
- Check X-Ray sampling rules: `aws xray get-sampling-rules`
- Verify the application is not running its own X-Ray daemon (conflicts with App Runner's managed daemon)
- Check X-Ray groups and filter expressions for trace visibility

## Phase 2 — Remediate

MUST:
- Create an observability configuration with tracing enabled if none exists
- Attach the observability configuration to the service via update-service
- Instrument the application with X-Ray SDK or OpenTelemetry SDK
- Do NOT include the X-Ray daemon in the container — App Runner manages it

SHOULD:
- Add xray:PutTraceSegments and xray:PutTelemetryRecords to the instance role
- Use automatic instrumentation where available (e.g., AWS X-Ray SDK auto-instrumentation for Node.js, Python, Java)
- Configure meaningful service names and annotations in traces

MAY:
- Set up custom sampling rules to control trace volume and cost
- Create X-Ray groups for filtering traces by service or error type
- Integrate with AWS ServiceLens for combined metrics, logs, and traces

## Common Issues

- symptoms: "No traces appearing in X-Ray console"
  diagnosis: "Observability configuration not attached to the service or tracing not enabled."
  resolution: "Create an observability configuration with TraceConfiguration Vendor=AWSXRAY and attach to the service."

- symptoms: "Traces appear but are incomplete or missing segments"
  diagnosis: "Application not fully instrumented — only some HTTP calls are traced."
  resolution: "Add X-Ray SDK instrumentation for all outbound HTTP calls, AWS SDK calls, and database queries."

- symptoms: "X-Ray daemon port conflict errors in logs"
  diagnosis: "Application container includes its own X-Ray daemon, conflicting with App Runner's managed daemon."
  resolution: "Remove the X-Ray daemon from the container. App Runner manages the daemon automatically."

## Output Format

```yaml
root_cause: "xray_tracing — <specific_cause>"
evidence:
  - type: observability_config
    content: "<observability configuration details>"
  - type: trace_data
    content: "<X-Ray trace summary>"
  - type: application_instrumentation
    content: "<SDK instrumentation status>"
severity: LOW
mitigation:
  immediate: "Enable observability configuration and instrument the application"
  long_term: "Implement comprehensive tracing with custom annotations and sampling rules"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Create observability configuration | GREEN | New resource creation |
| Attach observability config to service | YELLOW | Service configuration change |
| Instrument app with X-Ray SDK | YELLOW | Application code change |
| Add X-Ray permissions to instance role | YELLOW | IAM policy change |
| Set custom sampling rules | GREEN | Sampling configuration |

## Escalation Conditions

- Fix requires redeployment of production service
- NEVER suggest scaling to 0 (not supported)

## Data Sensitivity

- **Observability config**: Tracing configuration details — INTERNAL
- **Trace data**: Request paths, latencies, service map — INTERNAL
- **Instance role**: X-Ray IAM permissions — INTERNAL

## Prohibited Actions

- NEVER suggest hardcoding credentials in environment variables
- NEVER suggest scaling to 0 instances (not supported by App Runner)
- NEVER include the X-Ray daemon in the container (App Runner manages it)

## Phase 3 — Rollback

1. Detach observability configuration from the service
2. Revert X-Ray SDK instrumentation in application code
3. Remove X-Ray IAM permissions from instance role if no longer needed
4. Redeploy with the reverted configuration
5. Verify application performance is not impacted after removing tracing

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

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
