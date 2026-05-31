---
title: "B1 — SDK Instrumentation Issues"
description: "Diagnose X-Ray SDK instrumentation problems"
status: active
severity: HIGH
triggers:
  - "SDK not working"
  - "instrumentation error"
  - "X-Ray SDK issue"
  - "cannot instrument"
owner: devops-agent
objective: "Identify and resolve X-Ray SDK instrumentation issues"
context: "The X-Ray SDK must be properly integrated into application code to capture segments and subsegments. Issues include incorrect SDK initialization, missing client wrapping, version incompatibilities, and conflicts with other tracing libraries."
---

## Phase 1 — Triage

MUST:
- Verify the X-Ray SDK is included in the application dependencies
- Check application logs for X-Ray SDK errors or warnings
- Verify the SDK is initialized at application startup
- Check if AWS SDK clients are wrapped with X-Ray: look for captureAWS/captureAWSClient calls

SHOULD:
- Verify the SDK version is compatible with the runtime
- Check if HTTP clients are instrumented for outgoing calls
- Verify SQL clients are instrumented if database calls are expected in traces

MAY:
- Enable X-Ray SDK debug logging for detailed diagnostics
- Check for conflicts with other tracing libraries (OpenTelemetry, Datadog)

## Phase 2 — Remediate

MUST:
- Install the correct X-Ray SDK for the runtime (Node.js, Python, Java, .NET, Go, Ruby)
- Initialize the SDK at application startup before other imports
- Wrap AWS SDK clients with X-Ray instrumentation

SHOULD:
- Instrument HTTP clients for outgoing HTTP calls
- Add custom subsegments for important business logic
- Configure the SDK to connect to the daemon on the correct address

MAY:
- Consider migrating to OpenTelemetry with ADOT for vendor-neutral instrumentation
- Add annotations and metadata for trace searchability

## Common Issues

- symptoms: "SDK imported but no traces generated"
  diagnosis: "SDK not initialized or initialized after AWS SDK clients are created."
  resolution: "Initialize X-Ray SDK before creating any AWS SDK clients."

- symptoms: "SegmentNotFoundException in application logs"
  diagnosis: "Code trying to create subsegments outside of an active segment context."
  resolution: "Ensure a parent segment exists. Use AWSXRay.captureAsyncFunc for async operations."

- symptoms: "Traces show service but no downstream calls"
  diagnosis: "AWS SDK clients not wrapped with X-Ray instrumentation."
  resolution: "Wrap clients: const AWS = AWSXRay.captureAWS(require('aws-sdk'))."

## Output Format

```yaml
root_cause: "sdk_instrumentation_issue — <specific_cause>"
evidence:
  - type: sdk_version
    content: "<SDK version and runtime>"
  - type: initialization
    content: "<SDK initialization status>"
  - type: client_wrapping
    content: "<AWS SDK client instrumentation>"
severity: HIGH
mitigation:
  immediate: "Fix SDK instrumentation in application code"
  long_term: "Standardize instrumentation patterns across services"
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
