---
title: "B2 — Input Schema Issues"
description: "Diagnose input definition and schema mismatch problems"
status: active
severity: MEDIUM
triggers:
  - "attribute not found"
  - "schema mismatch"
  - "input definition error"
owner: devops-agent
objective: "Identify and resolve input schema issues"
context: "Schema issues occur when: attribute paths don't match payload, nested objects not properly defined, or data types incompatible."
---

## Phase 1 — Triage

MUST:
- Check input definition: `aws iotevents describe-input --input-name <name>`
- Compare attribute paths with actual payload structure
- Verify attribute names are case-sensitive matches
- Check for nested object path notation

SHOULD:
- Validate sample payload against input definition
- Check if extra attributes are causing issues (they shouldn't)
- Verify payload is valid JSON

MAY:
- Test with minimal payload containing only required attributes
- Check input creation date vs payload format changes

## Phase 2 — Remediate

MUST:
- Update input definition to match payload structure
- Fix attribute path notation for nested objects
- Ensure case-sensitive attribute name matching

SHOULD:
- Document expected payload format
- Version input definitions alongside payload producers

## Output Format

```yaml
root_cause: "input_schema — <specific_cause>"
evidence:
  - type: schema_comparison
    content: "<input definition vs payload>"
severity: MEDIUM
mitigation:
  immediate: "Fix input definition or payload"
  long_term: "Implement schema validation"
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
