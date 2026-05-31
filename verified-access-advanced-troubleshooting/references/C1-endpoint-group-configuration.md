---
title: "C1 — Endpoint Group Configuration"
description: "Diagnose Verified Access endpoint group configuration issues"
status: active
severity: MEDIUM
triggers: ["endpoint group error", "cannot create endpoint group", "group policy not applying", "endpoint group misconfigured"]
owner: devops-agent
objective: "Resolve Verified Access endpoint group creation and configuration issues"
context: "Endpoint groups organize endpoints under a Verified Access instance and hold Cedar policies. Each group can have its own policy controlling access to all endpoints within it."
---

## Phase 1 — Triage

MUST:
- List endpoint groups: `aws ec2 describe-verified-access-groups --query 'VerifiedAccessGroups[*].{Id:VerifiedAccessGroupId,InstanceId:VerifiedAccessInstanceId,Description:Description}'`
- Check group policy: `aws ec2 get-verified-access-group-policy --verified-access-group-id <group-id>`
- List endpoints in the group: `aws ec2 describe-verified-access-endpoints --query 'VerifiedAccessEndpoints[?VerifiedAccessGroupId==\`<group-id>\`].{Id:VerifiedAccessEndpointId,Status:Status}'`

SHOULD:
- Verify the group is associated with the correct Verified Access instance
- Check if the group has a policy document or is using default deny
- Review endpoint status within the group

MAY:
- Compare group configurations across environments
- Check CloudTrail for group modification events

## Phase 2 — Remediate

MUST:
- Ensure group is associated with the correct Verified Access instance
- Apply a Cedar policy to the group (default is deny-all without a policy)
- Verify endpoints are in active status within the group

SHOULD:
- Use descriptive group names and tags for management
- Test Cedar policy before applying to production groups

MAY:
- Implement group-per-application pattern for granular access control
- Use IaC for consistent group configuration across environments

## Common Issues

- symptoms: "All endpoints in group deny access despite no explicit forbid policy"
  diagnosis: "No Cedar policy applied to the group. Default behavior is deny-all."
  resolution: "Apply a Cedar permit policy: aws ec2 modify-verified-access-group-policy --verified-access-group-id <id> --policy-enabled --policy-document '<cedar-policy>'."

## Safety Ratings

```
safety_ratings:
  - "describe-verified-access-groups: GREEN — read-only enumeration"
  - "get-verified-access-group-policy: GREEN — read-only policy retrieval"
  - "modify-verified-access-group-policy: YELLOW — changes authorization rules"
  - "delete-verified-access-group: RED — removes group and disassociates endpoints"
```

## Escalation Conditions

- Group policy changes cause widespread access disruption
- Cannot apply policy due to Cedar syntax validation errors
- Group is in an error state that cannot be resolved
- Multiple groups need coordinated policy updates

## Data Sensitivity

- `aws ec2 describe-verified-access-groups`: LOW — returns group metadata
- Cedar policy content: MEDIUM — reveals authorization logic
- Endpoint associations: LOW — reveals application topology
- CloudTrail events: MEDIUM — reveals who modified group configurations

## Prohibited Actions

- NEVER delete an endpoint group with active endpoints without migrating them first
- NEVER apply untested Cedar policies to production groups
- NEVER remove the policy from a group without understanding the default deny behavior
- NEVER modify group-instance associations without understanding the impact on endpoints

## Phase 3 — Rollback

1. Revert Cedar policy to previous version: `aws ec2 modify-verified-access-group-policy --verified-access-group-id <id> --policy-document "<previous-policy>"`
2. If group was deleted, recreate it and reassociate endpoints
3. Verify all endpoints in the group are functioning after rollback
4. Check access logs to confirm policy evaluation is correct
5. Document the configuration change and rollback

## Output Format

```yaml
root_cause: "endpoint_group — <specific_cause>"
evidence:
  - type: group_config
    content: "<group details and policy>"
severity: MEDIUM
mitigation:
  immediate: "Fix group configuration"
  long_term: "Implement policy versioning and testing"
```

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
