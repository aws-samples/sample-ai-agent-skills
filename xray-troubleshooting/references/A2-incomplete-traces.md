---
title: "A2 — Incomplete Traces"
description: "Diagnose why traces have missing segments or subsegments"
status: active
severity: MEDIUM
triggers:
  - "incomplete trace"
  - "missing segments"
  - "missing subsegments"
  - "trace gaps"
owner: devops-agent
objective: "Identify why traces are incomplete and add missing instrumentation"
context: "Incomplete traces show some services but not others. This is usually caused by missing SDK instrumentation for downstream calls, trace ID not being propagated, or services not being instrumented at all."
---

## Phase 1 — Triage

MUST:
- Get the full trace: `aws xray batch-get-traces --trace-ids <trace-id>`
- Identify which segments and subsegments are present vs expected
- Check if the trace ID header is being propagated between services
- Verify SDK instrumentation in the application code for downstream calls

SHOULD:
- Check if AWS SDK calls are instrumented (DynamoDB, S3, SQS, etc.)
- Verify HTTP client calls are wrapped with X-Ray SDK
- Check for segment document size (64KB limit may cause truncation)

MAY:
- Enable debug logging in the X-Ray SDK to see what's being captured
- Check if subsegments are being created but not sent (daemon issue)

## Phase 2 — Remediate

MUST:
- Instrument AWS SDK calls with X-Ray SDK: wrap the AWS SDK client
- Propagate the X-Amzn-Trace-Id header in HTTP calls between services
- Ensure all services in the call chain have X-Ray enabled

SHOULD:
- Instrument HTTP clients for external API calls
- Add custom subsegments for important business logic
- Keep segment documents under 64KB

MAY:
- Use OpenTelemetry auto-instrumentation for broader coverage
- Add annotations for searchable trace metadata

## Common Issues

- symptoms: "AWS SDK calls not showing as subsegments"
  diagnosis: "AWS SDK client not instrumented with X-Ray SDK."
  resolution: "Wrap the AWS SDK client with X-Ray: AWSXRay.captureAWSClient(client)."

- symptoms: "Downstream service appears as separate trace"
  diagnosis: "Trace ID header not propagated between services."
  resolution: "Forward the X-Amzn-Trace-Id header in HTTP requests between services."

- symptoms: "Trace truncated with missing subsegments"
  diagnosis: "Segment document exceeds 64KB limit."
  resolution: "Reduce annotations/metadata size. Limit subsegment depth."

## Output Format

```yaml
root_cause: "incomplete_trace — <specific_cause>"
evidence:
  - type: trace_segments
    content: "<segments and subsegments present>"
  - type: missing_components
    content: "<expected but missing segments>"
  - type: instrumentation
    content: "<SDK instrumentation status>"
severity: MEDIUM
mitigation:
  immediate: "Add missing SDK instrumentation"
  long_term: "Standardize instrumentation across all services"
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
