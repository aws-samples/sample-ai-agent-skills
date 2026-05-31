---
title: "D1 — Thing Group Issues"
description: "Diagnose thing group creation, membership, and dynamic group query failures"
status: active
severity: MEDIUM
triggers: ["thing group empty", "dynamic group not populating", "cannot add thing to group", "group hierarchy error"]
owner: devops-agent
objective: "Resolve thing group configuration and membership issues"
context: "Thing groups organize devices into logical collections. Static groups have explicit membership. Dynamic groups use fleet indexing queries. Groups support hierarchy up to 7 levels deep."
---

## Phase 1 — Triage

MUST:
- List thing groups: `aws iot list-thing-groups --query 'thingGroups[*].{Name:groupName,Arn:groupArn}'`
- Describe the problematic group: `aws iot describe-thing-group --thing-group-name <name>`
- Check group membership: `aws iot list-things-in-thing-group --thing-group-name <name>`

SHOULD:
- For dynamic groups, check the query string: `aws iot describe-thing-group --thing-group-name <name> --query 'queryString'`
- Verify fleet indexing is enabled: `aws iot get-indexing-configuration`
- Check if thing has expected attributes: `aws iot describe-thing --thing-name <name> --query '{Attributes:attributes,ThingTypeName:thingTypeName}'`

MAY:
- Test dynamic group query directly: `aws iot search-index --query-string "<query>"`
- Check group hierarchy: `aws iot list-thing-groups-for-thing --thing-name <name>`
- Review CloudTrail for group modification events

## Phase 2 — Remediate

MUST:
- For dynamic groups, ensure fleet indexing is enabled with thing indexing mode REGISTRY_AND_SHADOW or REGISTRY
- Fix query string syntax for dynamic groups (uses AWS IoT query syntax, not SQL)
- Ensure things have the attributes referenced in dynamic group queries

SHOULD:
- Verify group hierarchy does not exceed 7 levels
- Check that parent groups exist before creating child groups
- Ensure thing attributes are indexed if used in dynamic group queries

MAY:
- Rebuild fleet index if stale: `aws iot update-indexing-configuration --thing-indexing-configuration thingIndexingMode=REGISTRY_AND_SHADOW`
- Create static group as fallback while debugging dynamic group queries
- Use thing group tags for additional organization

## Safety Ratings

```
safety_ratings:
  - "list-thing-groups: GREEN — read-only enumeration"
  - "describe-thing-group: GREEN — read-only inspection"
  - "list-things-in-thing-group: GREEN — read-only membership query"
  - "add-thing-to-thing-group: YELLOW — modifies group membership, affects policy inheritance"
  - "update-indexing-configuration: YELLOW — changes indexing mode, may trigger reindexing"
```

## Escalation Conditions

- Dynamic group query returns incorrect results affecting policy attachment
- Group hierarchy changes impact inherited policies across large device fleets
- Fleet indexing reindexing takes too long for large fleets (100k+ things)
- Group membership changes trigger unintended job executions

## Data Sensitivity

- `aws iot describe-thing-group`: LOW — returns group metadata and query string
- `aws iot list-things-in-thing-group`: MEDIUM — returns device identifiers
- `aws iot search-index`: MEDIUM — returns device attributes and connectivity
- `aws iot get-indexing-configuration`: LOW — returns indexing settings

## Prohibited Actions

- NEVER delete a thing group that has policies attached without understanding policy inheritance impact
- NEVER modify dynamic group queries in production without testing the query first via search-index
- NEVER disable fleet indexing if dynamic groups depend on it — groups will become empty
- NEVER create circular group hierarchies (not possible via API but avoid confusing structures)

## Phase 3 — Rollback

1. If group was deleted, recreate with same name and re-add members: `aws iot create-thing-group --thing-group-name <name>`
2. If dynamic query was changed, restore previous query: `aws iot update-dynamic-thing-group --thing-group-name <name> --query-string "<previous-query>"`
3. If indexing was modified, restore previous configuration
4. Re-attach policies to group if they were detached: `aws iot attach-policy --policy-name <policy> --target <group-arn>`
5. Verify group membership matches expected state

## Common Issues

- symptoms: "Dynamic thing group shows 0 members despite matching things existing"
  diagnosis: "Fleet indexing is not enabled or indexing mode does not include the attributes used in the query."
  resolution: "Enable fleet indexing with REGISTRY_AND_SHADOW mode. Wait for index to build (may take minutes to hours for large fleets)."

- symptoms: "Thing added to group but does not inherit group policy"
  diagnosis: "Policy is attached to a parent group but thing is in a child group, or policy attachment is on the thing group ARN not the group name."
  resolution: "Verify policy is attached to the correct group ARN. Check group hierarchy for policy inheritance path."

## Output Format

```yaml
root_cause: "thing_group — <specific_cause>"
evidence:
  - type: group_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix group configuration"
  long_term: "Implement group membership monitoring and query validation"
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
