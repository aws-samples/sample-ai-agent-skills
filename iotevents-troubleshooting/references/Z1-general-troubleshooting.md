---
title: "Z1 — General IoT Events Troubleshooting"
description: "Catch-all runbook for unclassified IoT Events issues"
status: active
severity: MEDIUM
triggers:
  - "IoT Events not working"
  - "something wrong with IoT Events"
owner: devops-agent
objective: "Systematically triage unclassified IoT Events issues"
context: "Use this runbook when the issue doesn't clearly match a specific category."
---

## Phase 1 — Triage

MUST:
- List detector models: `aws iotevents list-detector-models`
- List inputs: `aws iotevents list-inputs`
- List alarm models: `aws iotevents list-alarm-models`
- Check CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=iotevents.amazonaws.com --max-results 20`

SHOULD:
- Check CloudWatch IoT Events metrics
- Verify IAM permissions
- Check service health

MAY:
- List detector instances for active models
- Check Service Quotas usage

## Phase 2 — Route to specific runbook

Based on findings, route to A1-A2, B1-B2, C1-C2, D1-D2, or E1-E2.

## Output Format

```yaml
root_cause: "general — <routed_to_specific_runbook>"
evidence:
  - type: triage_summary
    content: "<initial findings>"
severity: MEDIUM
mitigation:
  immediate: "Follow specific runbook"
  long_term: "Implement comprehensive monitoring"
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
