---
title: "D2 — Thing Group Issues"
description: "Diagnose thing group membership and policy inheritance problems"
status: active
severity: MEDIUM
triggers:
  - "thing group error"
  - "group membership"
  - "dynamic group not working"
  - "group policy not applied"
  - "thing not in group"
owner: devops-agent
objective: "Identify and resolve thing group issues"
context: "Thing group issues include: dynamic group query not matching, fleet indexing not enabled, group policy not applying, nested group inheritance, or group membership limits."
---

## Phase 1 — Triage

MUST:
- Check thing groups: `aws iot list-thing-groups`
- Check thing's group membership: `aws iot list-thing-groups-for-thing --thing-name <thing-name>`
- Check group details: `aws iot describe-thing-group --thing-group-name <group>`
- For dynamic groups: verify fleet indexing is enabled: `aws iot get-indexing-configuration`

SHOULD:
- Check policies attached to group: `aws iot list-attached-policies --target <group-arn>`
- Verify dynamic group query syntax
- Check nested group hierarchy

MAY:
- Test dynamic group query: `aws iot search-index --query-string "<query>"`
- Check group membership limits

## Phase 2 — Remediate

MUST:
- Enable fleet indexing for dynamic groups: `aws iot update-indexing-configuration --thing-indexing-configuration thingIndexingMode=REGISTRY_AND_SHADOW`
- Fix dynamic group query syntax
- Add thing to static group: `aws iot add-thing-to-thing-group --thing-group-name <group> --thing-name <thing>`

SHOULD:
- Attach policy to group: `aws iot attach-policy --policy-name <policy> --target <group-arn>`
- Create dynamic group: `aws iot create-dynamic-thing-group --thing-group-name <name> --query-string "attributes.type:sensor"`

MAY:
- Implement group hierarchy for organizational structure
- Use group policies for fleet-wide permissions

## Output Format

```yaml
root_cause: "thing_group_issue — <specific_cause>"
evidence:
  - type: group_config
    content: "<group details>"
  - type: indexing_config
    content: "<fleet indexing status>"
severity: MEDIUM
mitigation:
  immediate: "Fix group membership or enable fleet indexing"
  long_term: "Design group hierarchy with proper policy inheritance"
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
