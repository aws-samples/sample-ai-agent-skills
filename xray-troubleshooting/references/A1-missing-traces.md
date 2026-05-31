---
title: "A1 — Missing Traces"
description: "Diagnose why traces are not appearing in X-Ray"
status: active
severity: HIGH
triggers:
  - "traces not appearing"
  - "no traces in X-Ray"
  - "missing traces"
  - "X-Ray empty"
owner: devops-agent
objective: "Identify why traces are not being captured and resolve the issue"
context: "Missing traces can be caused by sampling rules filtering them out, daemon not running, IAM permissions missing, SDK not instrumented, or tracing not enabled on the service. Multiple layers must be checked."
---

## Phase 1 — Triage

MUST:
- Check if tracing is enabled on the service (Lambda, API Gateway, etc.)
- Verify sampling rules: `aws xray get-sampling-rules`
- Check sampling statistics: `aws xray get-sampling-statistic-summaries`
- Verify IAM permissions for the service role: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names xray:PutTraceSegments xray:PutTelemetryRecords`

SHOULD:
- Check if the X-Ray daemon is running (for non-Lambda services)
- Verify the time range in the X-Ray console matches when requests were made
- Check CloudWatch metrics for the service to confirm requests are occurring
- Look for X-Ray-related errors in application logs

MAY:
- Test with a sampling rule that captures 100% temporarily
- Check if the service is in a VPC that blocks UDP 2000 to the daemon

## Phase 2 — Remediate

MUST:
- Enable tracing on the service if not enabled
- Fix IAM permissions: add AWSXRayDaemonWriteAccess managed policy to the service role
- Ensure the X-Ray daemon is running and reachable on UDP 2000

SHOULD:
- Review and fix sampling rules to ensure desired traffic is sampled
- Instrument application code with X-Ray SDK if not already done
- Verify the X-Amzn-Trace-Id header is being propagated

MAY:
- Create a custom sampling rule for the specific service
- Set up CloudWatch alarms for X-Ray error metrics

## Common Issues

- symptoms: "Lambda traces missing"
  diagnosis: "Active tracing not enabled or execution role missing X-Ray permissions."
  resolution: "Enable active tracing and add xray:PutTraceSegments permission to the role."

- symptoms: "ECS traces missing"
  diagnosis: "X-Ray daemon sidecar not running in the task definition."
  resolution: "Add X-Ray daemon as a sidecar container in the ECS task definition."

- symptoms: "Traces appear for some requests but not others"
  diagnosis: "Sampling rules filtering out some requests."
  resolution: "Review sampling rules and priorities. Adjust rates or create specific rules."

## Output Format

```yaml
root_cause: "missing_traces — <specific_cause>"
evidence:
  - type: tracing_enabled
    content: "<tracing configuration on the service>"
  - type: sampling_rules
    content: "<active sampling rules and priorities>"
  - type: iam_permissions
    content: "<X-Ray permissions on the service role>"
severity: HIGH
mitigation:
  immediate: "Enable tracing and fix permissions"
  long_term: "Standardize X-Ray configuration across services"
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
