---
title: "C1 — Lambda Tracing Issues"
description: "Diagnose X-Ray tracing problems with AWS Lambda functions"
status: active
severity: HIGH
triggers:
  - "Lambda traces missing"
  - "Lambda X-Ray not working"
  - "function tracing"
  - "active tracing Lambda"
owner: devops-agent
objective: "Identify and resolve Lambda X-Ray tracing issues"
context: "Lambda has built-in X-Ray integration (no daemon needed). Active tracing must be enabled on the function. The execution role needs X-Ray permissions. Downstream calls still require SDK instrumentation in the function code."
---

## Phase 1 — Triage

MUST:
- Check if active tracing is enabled: `aws lambda get-function-configuration --function-name <function> --query 'TracingConfig'`
- Verify execution role has X-Ray permissions: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names xray:PutTraceSegments xray:PutTelemetryRecords`
- Check for traces in the time range: `aws xray get-trace-summaries --start-time <start> --end-time <end> --filter-expression 'service("<function-name>")'`
- Verify the function is being invoked: `aws cloudwatch get-metric-statistics --namespace AWS/Lambda --metric-name Invocations --dimensions Name=FunctionName,Value=<function> --start-time <start> --end-time <end> --period 300 --statistics Sum`

SHOULD:
- Check if the function code has X-Ray SDK instrumentation for downstream calls
- Verify Lambda layers include X-Ray SDK if used
- Check function logs for X-Ray-related errors

MAY:
- Test with a simple function that only creates a segment
- Check if the function is in a VPC that might affect X-Ray connectivity

## Phase 2 — Remediate

MUST:
- Enable active tracing: `aws lambda update-function-configuration --function-name <function> --tracing-config Mode=Active`
- Add X-Ray permissions to the execution role: attach AWSXRayDaemonWriteAccess policy
- Instrument downstream calls in function code with X-Ray SDK

SHOULD:
- Use X-Ray SDK to wrap AWS SDK clients in the function code
- Add annotations for searchable trace metadata
- Configure sampling rules specific to the Lambda function

MAY:
- Use Lambda layers for X-Ray SDK to reduce deployment package size
- Consider OpenTelemetry Lambda layer as an alternative

## Common Issues

- symptoms: "Active tracing enabled but no traces"
  diagnosis: "Execution role missing xray:PutTraceSegments permission."
  resolution: "Attach AWSXRayDaemonWriteAccess to the execution role."

- symptoms: "Lambda segment appears but no downstream calls"
  diagnosis: "Function code not instrumented with X-Ray SDK."
  resolution: "Add X-Ray SDK and wrap AWS SDK clients in the function code."

- symptoms: "Traces appear for some invocations but not all"
  diagnosis: "Sampling rules filtering out some invocations."
  resolution: "Check sampling rules. Lambda uses the X-Ray sampling rules, not its own."

## Output Format

```yaml
root_cause: "lambda_tracing_issue — <specific_cause>"
evidence:
  - type: tracing_config
    content: "<Lambda tracing configuration>"
  - type: iam_permissions
    content: "<X-Ray permissions on execution role>"
  - type: invocation_count
    content: "<Lambda invocation metrics>"
severity: HIGH
mitigation:
  immediate: "Enable tracing and fix permissions"
  long_term: "Standardize Lambda tracing configuration via IaC"
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
