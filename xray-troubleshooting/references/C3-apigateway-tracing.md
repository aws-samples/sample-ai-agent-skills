---
title: "C3 — API Gateway Tracing Issues"
description: "Diagnose X-Ray tracing problems with API Gateway"
status: active
severity: MEDIUM
triggers:
  - "API Gateway traces missing"
  - "API Gateway X-Ray"
  - "REST API tracing"
  - "HTTP API tracing"
owner: devops-agent
objective: "Identify and resolve API Gateway X-Ray tracing issues"
context: "API Gateway supports X-Ray tracing for REST APIs and HTTP APIs. Tracing must be enabled per stage. API Gateway automatically propagates the trace ID header to downstream integrations. Tracing adds latency overhead."
---

## Phase 1 — Triage

MUST:
- Check if tracing is enabled on the stage: `aws apigateway get-stage --rest-api-id <api-id> --stage-name <stage> --query 'tracingEnabled'`
- Verify traces exist: `aws xray get-trace-summaries --start-time <start> --end-time <end> --filter-expression 'service("<api-name>")'`
- Check API Gateway execution logs for errors
- Verify the API is receiving requests: check CloudWatch metrics

SHOULD:
- Check if the downstream integration (Lambda, HTTP) is also instrumented
- Verify the X-Amzn-Trace-Id header is being propagated
- Check sampling rules that may affect API Gateway traces

MAY:
- Enable detailed CloudWatch logging on the stage for debugging
- Check if the API Gateway execution role has X-Ray permissions

## Phase 2 — Remediate

MUST:
- Enable tracing on the stage: `aws apigateway update-stage --rest-api-id <api-id> --stage-name <stage> --patch-operations op=replace,path=/tracingEnabled,value=true`
- Ensure downstream integrations are also instrumented for end-to-end tracing

SHOULD:
- Configure appropriate sampling rules for the API
- Enable tracing on all stages that need monitoring

MAY:
- Consider the latency overhead of tracing in production
- Set up X-Ray groups for API Gateway traces

## Common Issues

- symptoms: "API Gateway stage shows tracing enabled but no traces"
  diagnosis: "Sampling rules filtering out all requests, or no traffic to the API."
  resolution: "Check sampling rules. Verify the API is receiving requests."

- symptoms: "API Gateway trace shows but downstream Lambda is missing"
  diagnosis: "Lambda function not configured for active tracing."
  resolution: "Enable active tracing on the Lambda function and add X-Ray permissions."

- symptoms: "HTTP API tracing not available"
  diagnosis: "HTTP APIs have limited X-Ray support compared to REST APIs."
  resolution: "Use REST API for full X-Ray integration, or instrument the backend directly."

## Output Format

```yaml
root_cause: "apigateway_tracing_issue — <specific_cause>"
evidence:
  - type: tracing_config
    content: "<stage tracing configuration>"
  - type: trace_summaries
    content: "<traces found for the API>"
  - type: downstream_integration
    content: "<downstream service tracing status>"
severity: MEDIUM
mitigation:
  immediate: "Enable tracing on the API Gateway stage"
  long_term: "Standardize tracing across API Gateway and downstream services"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Trace data reveals application request flow and dependencies
- Segment metadata may contain request/response details
- Service maps expose complete application architecture
- Classification: CONFIDENTIAL - application architecture and request flow data

## Prohibited Actions

- NEVER modify sampling rules that could cause production trace data loss
- NEVER delete X-Ray groups used by production monitoring dashboards
- NEVER change encryption settings without coordinating with security team
- NEVER disable tracing on production services without approval

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

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
