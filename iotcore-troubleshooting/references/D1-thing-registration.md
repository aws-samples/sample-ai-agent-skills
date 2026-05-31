---
title: "D1 — Thing Registration Issues"
description: "Diagnose thing registration and provisioning problems"
status: active
severity: MEDIUM
triggers:
  - "thing creation failed"
  - "thing not found"
  - "registration error"
  - "provisioning failed"
  - "thing type error"
owner: devops-agent
objective: "Identify and resolve thing registration issues"
context: "Thing registration issues include: IAM permissions for CreateThing, thing type deprecation, attribute validation, thing name conflicts, or fleet provisioning template errors."
---

## Phase 1 — Triage

MUST:
- Check if thing exists: `aws iot describe-thing --thing-name <thing-name>`
- Check thing type: `aws iot describe-thing-type --thing-type-name <type>`
- Verify IAM permissions for iot:CreateThing
- Check for thing name conflicts or invalid characters

SHOULD:
- Check fleet provisioning templates: `aws iot list-provisioning-templates`
- Verify thing type is not deprecated
- Check thing attributes against type definition

MAY:
- Review CloudTrail for registration failures
- Check bulk registration job status

## Phase 2 — Remediate

MUST:
- Create thing with proper attributes: `aws iot create-thing --thing-name <name> --thing-type-name <type> --attribute-payload '{"attributes":{"key":"value"}}'`
- Fix IAM permissions if access denied
- Use valid thing names (alphanumeric, hyphens, underscores, colons)

SHOULD:
- Use fleet provisioning for automated registration
- Define thing types for consistent attribute schemas

MAY:
- Implement just-in-time provisioning (JITP)
- Use bulk registration for large fleets

## Output Format

```yaml
root_cause: "thing_registration_issue — <specific_cause>"
evidence:
  - type: thing_status
    content: "<thing details or error>"
  - type: permissions
    content: "<IAM permissions>"
severity: MEDIUM
mitigation:
  immediate: "Fix registration error or permissions"
  long_term: "Implement automated provisioning workflow"
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
- Never revoke or delete device certificates without confirming device decommission
- Never modify IoT policies on production devices without testing in a staging environment
- Never delete thing groups that contain active production devices
- Never alter MQTT topic rules that route critical telemetry data
- Never disable custom authorizers serving active device connections

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Reattach previous certificates if device connectivity is lost after changes
- Restore previous IoT policy versions if devices lose permissions
- Revert rule engine action configurations if message routing breaks

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
