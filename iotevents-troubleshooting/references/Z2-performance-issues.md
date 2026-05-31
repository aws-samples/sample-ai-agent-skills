---
title: "Z2 — Performance Issues"
description: "Diagnose IoT Events throughput and latency problems"
status: active
severity: MEDIUM
triggers:
  - "slow processing"
  - "high latency"
  - "throttling"
  - "messages delayed"
owner: devops-agent
objective: "Identify and resolve IoT Events performance issues"
context: "Performance issues occur when: input rate exceeds limits, too many detector instances, complex expressions, or action targets slow."
---

## Phase 1 — Triage

MUST:
- Check input processing rate vs account limits
- Count active detector instances: `aws iotevents-data list-detectors --detector-model-name <name>`
- Check CloudWatch for throttling metrics
- Verify action target response times

SHOULD:
- Check expression complexity in detector model
- Verify batch input sizes
- Check Service Quotas for current limits

MAY:
- Profile action execution times
- Check for unnecessary detector instances

## Phase 2 — Remediate

MUST:
- Reduce input rate or request quota increase
- Clean up unused detector instances
- Simplify complex expressions

SHOULD:
- Batch inputs efficiently
- Optimize action targets for faster response

## Output Format

```yaml
root_cause: "performance — <specific_cause>"
evidence:
  - type: metrics
    content: "<throughput and latency data>"
severity: MEDIUM
mitigation:
  immediate: "Reduce load or increase quotas"
  long_term: "Optimize model and input patterns"
```

## Safety Ratings
```
safety_ratings:
  - "Triage commands (describe/list/get): GREEN — read-only"
  - "Configuration changes: YELLOW — recoverable"
  - "Delete/terminate operations: RED — irreversible"
```

## Escalation Conditions
- "Resource serves production workload"
- "Fix requires deleting or recreating resources"
- "Fix involves security or encryption changes"
- "Blast radius affects multiple dependent services"

## Data Sensitivity
- HIGH: credentials, PII, security configurations
- MEDIUM: resource configurations, metrics
- LOW: status information, error messages

## Prohibited Actions
- Never delete detector models that are actively monitoring production inputs
- Never modify alarm models during active alarm states without understanding impact
- Never change input routing configurations without verifying downstream detectors
- Never alter state transition logic on production detector models without testing
- Never remove action targets (Lambda, SNS, etc.) from active detector models

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Republish previous detector model version if event processing breaks
- Restore previous alarm model configuration if alarm evaluation fails
- Revert input routing changes if detectors stop receiving data

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-instances"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-console-output"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "ssm send-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound security group rules as a fix"
  - "NEVER suggest disabling instance metadata service"
  - "NEVER terminate instances without confirmation"
