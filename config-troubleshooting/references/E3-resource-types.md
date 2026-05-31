---
title: "E3 — Resource Type Recording Issues"
description: "Diagnose issues with AWS Config resource type recording coverage"
status: active
severity: MEDIUM
triggers:
  - "resource not recorded"
  - "resource type missing"
  - "not recording resource"
  - "global resources"
  - "third-party resources"
owner: devops-agent
objective: "Identify and resolve resource type recording gaps in AWS Config"
context: "Config can record all supported resource types or a specific subset. Issues arise when expected resource types are not included, global resources are not recorded, or third-party resources require custom schemas. Resource type coverage directly affects rule evaluation scope."
---

## Phase 1 — Triage

MUST:
- Check recorder configuration for resource types: `aws configservice describe-configuration-recorders`
- Verify if recording all resources or specific types
- Check if global resources (IAM) are included: look for `includeGlobalResourceTypes` in recorder config
- List discovered resource types: `aws configservice get-discovered-resource-counts`
- Check if the specific resource type is supported: `aws configservice list-discovered-resources --resource-type <type>`

SHOULD:
- Compare recorded types against rule scope requirements
- Check for third-party resource schemas: `aws cloudformation list-types --type RESOURCE --visibility PRIVATE`
- Verify the recorder has been running long enough to capture the resource type

MAY:
- Check AWS documentation for the latest supported resource types
- Verify resource type names match the exact AWS format (e.g., AWS::EC2::Instance)

## Phase 2 — Remediate

MUST:
- Add missing resource types to the recorder configuration
- Enable global resource recording if IAM rules are needed
- Restart the recorder after configuration changes

SHOULD:
- Avoid recording global resources in multiple regions (creates duplicates and extra cost)
- Document which resource types are recorded and why

MAY:
- Register third-party resource schemas for custom resource recording
- Use resource type filtering to optimize costs

## Common Issues

- symptoms: "Config rule shows NOT_APPLICABLE but resources of that type exist"
  diagnosis: "The recorder is not configured to record that resource type."
  resolution: "Add the resource type to the recorder configuration or enable 'Record all supported resource types.'"

- symptoms: "IAM-related rules show no compliance data"
  diagnosis: "Global resource recording (includeGlobalResourceTypes) is not enabled."
  resolution: "Enable includeGlobalResourceTypes in the recorder configuration. Only enable in one region to avoid duplicates."

- symptoms: "Custom or third-party resources not appearing in Config"
  diagnosis: "Third-party resources require a registered CloudFormation schema and explicit inclusion."
  resolution: "Register the resource type schema with CloudFormation and add it to the recorder configuration."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Add resource types to recorder | YELLOW | Configuration change; reversible |
| Enable global resource recording | YELLOW | Configuration change; reversible |
| Register third-party schemas | YELLOW | Schema registration; can be deregistered |

## Escalation Conditions

- Recorder configuration changes
- Global resource recording changes

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-configuration-recorders` | LOW | Recorder configuration |
| `get-discovered-resource-counts` | LOW | Aggregate resource counts |
| `list-discovered-resources` | MEDIUM | Resource IDs |

## Prohibited Actions

- NEVER suggest stopping Config recorder
- NEVER suggest deleting delivery channel
- NEVER suggest disabling organization rules

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Resource type addition | Revert recorder to previous resource type list |
| Global resource enablement | Disable global resource recording |

## Output Format

```yaml
root_cause: "resource_type_recording — <specific_cause>"
evidence:
  - type: recorder_config
    content: "<describe-configuration-recorders output showing resource types>"
  - type: discovered_resources
    content: "<discovered resource counts>"
  - type: rule_scope
    content: "<rule scope vs recorded types comparison>"
severity: MEDIUM
mitigation:
  immediate: "Add missing resource types and restart the recorder"
  long_term: "Maintain documentation of recorded resource types and review periodically"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

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
  - "NEVER suggest disabling Config rules to fix compliance"
  - "NEVER suggest deleting the configuration recorder"
  - "NEVER remove remediation actions without review"
