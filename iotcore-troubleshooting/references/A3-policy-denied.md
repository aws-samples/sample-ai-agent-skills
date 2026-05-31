---
title: "A3 — Policy Denied Errors"
description: "Diagnose IoT policy authorization failures"
status: active
severity: HIGH
triggers:
  - "policy denied"
  - "not authorized"
  - "publish denied"
  - "subscribe denied"
  - "authorization error"
owner: devops-agent
objective: "Identify and resolve IoT policy authorization issues"
context: "Policy denied errors occur when: IoT policy doesn't allow the action, topic ARN doesn't match, client ID restriction, or policy variables not resolving correctly."
---

## Phase 1 — Triage

MUST:
- Get policies attached to certificate: `aws iot list-attached-policies --target <cert-arn>`
- Check policy document: `aws iot get-policy --policy-name <policy-name>`
- Verify policy allows the specific action (iot:Publish, iot:Subscribe, iot:Receive)
- Verify topic ARN in policy matches the actual topic being used

SHOULD:
- Check for policy variables (${iot:Connection.Thing.ThingName}) resolution
- Verify client ID matches policy resource for iot:Connect
- Check if multiple policies are attached (union of permissions)
- Review IoT logs for specific denial reasons

MAY:
- Test policy with IoT policy simulator in console
- Check for thing group policies that may apply

## Phase 2 — Remediate

MUST:
- Update policy to allow required actions and topics
- Ensure topic ARN format is correct: `arn:aws:iot:<region>:<account>:topic/<topic-name>`
- For subscribe: use `topicfilter` resource type, not `topic`

SHOULD:
- Create comprehensive policy: `aws iot create-policy --policy-name <name> --policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Action":["iot:Connect"],"Resource":"arn:aws:iot:<region>:<account>:client/${iot:Connection.Thing.ThingName}"},{"Effect":"Allow","Action":["iot:Publish","iot:Receive"],"Resource":"arn:aws:iot:<region>:<account>:topic/devices/${iot:Connection.Thing.ThingName}/*"},{"Effect":"Allow","Action":"iot:Subscribe","Resource":"arn:aws:iot:<region>:<account>:topicfilter/devices/${iot:Connection.Thing.ThingName}/*"}]}'`

MAY:
- Use policy variables for dynamic per-device permissions
- Implement least-privilege policies

## Output Format

```yaml
root_cause: "policy_denied — <specific_cause>"
evidence:
  - type: policy_document
    content: "<IoT policy>"
  - type: denied_action
    content: "<action and topic that was denied>"
severity: HIGH
mitigation:
  immediate: "Update IoT policy to allow required actions"
  long_term: "Implement policy templates with proper topic scoping"
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
