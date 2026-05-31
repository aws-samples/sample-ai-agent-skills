---
title: "B1 — Input Routing Issues"
description: "Diagnose input message routing to detector models"
status: active
severity: HIGH
triggers:
  - "input not reaching detector"
  - "detector not receiving data"
  - "messages dropped"
owner: devops-agent
objective: "Identify and resolve input routing failures"
context: "Input routing fails when: input name wrong, payload doesn't match schema, key value missing, or BatchPutMessage API errors."
---

## Phase 1 — Triage

MUST:
- Verify input exists: `aws iotevents describe-input --input-name <name>`
- Check detector model references correct input name
- Verify payload matches input attribute definitions
- Test with BatchPutMessage: `aws iotevents-data batch-put-message --messages '[{"messageId":"test1","inputName":"<name>","payload":"<base64-json>"}]'`

SHOULD:
- Check if key value is present in payload for keyed detectors
- Verify IAM permissions for iotevents-data:BatchPutMessage
- Check CloudWatch for input processing metrics

MAY:
- Check IoT Core rule action if routing from IoT Core
- Verify payload size within limits

## Phase 2 — Remediate

MUST:
- Fix input name in detector model or API call
- Ensure payload contains all required attributes
- Include key value in payload for keyed detectors

SHOULD:
- Implement payload validation before sending
- Monitor input processing success rates

## Output Format

```yaml
root_cause: "input_routing — <specific_cause>"
evidence:
  - type: input_config
    content: "<input definition and payload>"
severity: HIGH
mitigation:
  immediate: "Fix input name or payload"
  long_term: "Implement input validation"
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
