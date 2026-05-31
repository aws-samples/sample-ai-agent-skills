---
title: "E2 — IoT Policy Configuration Issues"
description: "Diagnose IoT policy misconfiguration and permission problems"
status: active
severity: HIGH
triggers:
  - "IoT policy error"
  - "policy too permissive"
  - "policy variables"
  - "policy versioning"
  - "policy not working"
owner: devops-agent
objective: "Identify and resolve IoT policy configuration issues"
context: "IoT policy issues include: overly permissive policies, policy variables not resolving, version management, policy size limits, or incorrect resource ARN format."
---

## Phase 1 — Triage

MUST:
- Check policy document: `aws iot get-policy --policy-name <policy-name>`
- Check policy versions: `aws iot list-policy-versions --policy-name <policy-name>`
- Verify resource ARN format (topic vs topicfilter vs client)
- Check policy variables usage (${iot:Connection.Thing.ThingName})

SHOULD:
- Verify policy is not using wildcard (*) for all resources
- Check policy size (max 2048 characters)
- Review all policy versions and default version

MAY:
- Audit policies across all certificates
- Check for conflicting policies on same certificate

## Phase 2 — Remediate

MUST:
- Fix resource ARNs: use `topic/` for publish/receive, `topicfilter/` for subscribe, `client/` for connect
- Implement least-privilege policies
- Use policy variables for per-device scoping

SHOULD:
- Create versioned policy: `aws iot create-policy-version --policy-name <name> --policy-document file://policy.json --set-as-default`
- Remove overly permissive policies

MAY:
- Implement policy templates for different device types
- Use AWS IoT Device Defender for policy auditing

## Output Format

```yaml
root_cause: "iot_policy_issue — <specific_cause>"
evidence:
  - type: policy_document
    content: "<policy JSON>"
  - type: resource_arns
    content: "<ARN format check>"
severity: HIGH
mitigation:
  immediate: "Fix policy document and resource ARNs"
  long_term: "Implement least-privilege policy templates"
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
