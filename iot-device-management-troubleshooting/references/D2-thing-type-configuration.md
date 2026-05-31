---
title: "D2 — Thing Type Configuration"
description: "Diagnose thing type creation, deprecation, and migration issues"
status: active
severity: LOW
triggers: ["thing type error", "cannot modify thing type", "thing type deprecated", "attribute mismatch"]
owner: devops-agent
objective: "Resolve thing type configuration issues and manage type lifecycle"
context: "Thing types define searchable attributes for things. Types are immutable after creation. Deprecation is required before deletion with a 5-minute waiting period."
---

## Phase 1 — Triage

MUST:
- List thing types: `aws iot list-thing-types --query 'thingTypes[*].{Name:thingTypeName,Deprecated:thingTypeMetadata.deprecated}'`
- Describe the problematic type: `aws iot describe-thing-type --thing-type-name <name>`
- Check things using this type: `aws iot list-things --thing-type-name <name> --query 'things[*].thingName'`

SHOULD:
- Verify thing type attributes match expected schema
- Check if type is deprecated: `aws iot describe-thing-type --thing-type-name <name> --query 'thingTypeMetadata.deprecated'`
- Review if new things can still be created with this type

MAY:
- Compare attributes across thing types for migration planning
- Check CloudTrail for thing type modification events
- Review fleet indexing configuration for custom attribute indexing

## Phase 2 — Remediate

MUST:
- For attribute changes: create a new thing type with correct attributes
- Migrate things to new type: `aws iot update-thing --thing-name <name> --thing-type-name <new-type>`
- Deprecate old type after migration: `aws iot deprecate-thing-type --thing-type-name <old-type>`

SHOULD:
- Plan migration in batches to avoid disruption
- Update fleet indexing queries that reference old type attributes
- Update dynamic thing groups that filter by thing type name

MAY:
- Delete deprecated type after 5-minute waiting period: `aws iot delete-thing-type --thing-type-name <old-type>`
- Automate migration using a script that iterates over things of the old type
- Document the type change for device management team

## Safety Ratings

```
safety_ratings:
  - "list-thing-types: GREEN — read-only enumeration"
  - "describe-thing-type: GREEN — read-only inspection"
  - "update-thing (change type): YELLOW — modifies thing metadata, may affect queries"
  - "deprecate-thing-type: YELLOW — prevents new things from using this type"
  - "delete-thing-type: RED — permanently removes type definition"
```

## Escalation Conditions

- Large fleet (10k+ things) requires type migration affecting production operations
- Thing type change breaks existing fleet indexing queries or dynamic groups
- Deprecated type is still referenced by provisioning templates
- Type deletion blocked because things still reference it

## Data Sensitivity

- `aws iot list-thing-types`: LOW — returns type names and metadata
- `aws iot describe-thing-type`: LOW — returns type definition and attributes
- `aws iot list-things`: MEDIUM — returns device identifiers
- `aws iot update-thing`: LOW — modifies thing metadata only

## Prohibited Actions

- NEVER attempt to modify thing type attributes directly — types are immutable
- NEVER delete a thing type without first migrating all things to a new type
- NEVER deprecate a thing type referenced by active provisioning templates
- NEVER skip the 5-minute waiting period between deprecation and deletion

## Phase 3 — Rollback

1. If type was deprecated prematurely, undeprecate: `aws iot deprecate-thing-type --thing-type-name <name> --undo-deprecate`
2. If things were migrated to wrong type, migrate back: `aws iot update-thing --thing-name <name> --thing-type-name <original-type>`
3. If type was deleted, recreate with same attributes: `aws iot create-thing-type --thing-type-name <name> --thing-type-properties "thingTypeDescription=<desc>,searchableAttributes=<attrs>"`
4. Verify fleet indexing queries still return expected results
5. Confirm provisioning templates reference correct thing types

## Common Issues

- symptoms: "Cannot add new attribute to existing thing type"
  diagnosis: "Thing types are immutable. Attributes cannot be added, removed, or modified after creation."
  resolution: "Create a new thing type with the desired attributes. Migrate things to the new type. Deprecate the old type."

- symptoms: "DeleteThingType returns error about active things"
  diagnosis: "Things are still associated with the type. All things must be migrated or have their type removed before deletion."
  resolution: "Update all things to use a different type or remove type association. Then deprecate and delete."

## Output Format

```yaml
root_cause: "thing_type — <specific_cause>"
evidence:
  - type: type_error
    content: "<error details>"
severity: LOW
mitigation:
  immediate: "Create new type and migrate"
  long_term: "Plan thing type schema versioning strategy"
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
