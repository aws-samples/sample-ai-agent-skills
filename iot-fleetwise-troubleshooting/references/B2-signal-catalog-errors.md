---
title: "B2 — Signal Catalog Errors"
description: "Diagnose signal catalog creation, signal definition, and dependency issues"
status: active
severity: CRITICAL
triggers: ["signal catalog error", "signal not found", "catalog creation failed", "signal definition invalid", "duplicate signal"]
owner: devops-agent
objective: "Resolve signal catalog issues — the foundational resource for all FleetWise operations"
context: "Signal catalogs are account-level singletons (one per account per region). They define all available signals. All model manifests reference signals from this catalog. Changes affect all dependent resources."
---

## Phase 1 — Triage

MUST:
- List signal catalogs: `aws iotfleetwise list-signal-catalogs --query 'summaries[*].{Name:name,Arn:arn}'`
- Describe the catalog: `aws iotfleetwise get-signal-catalog --name <name>`
- Check catalog node counts: `aws iotfleetwise get-signal-catalog --name <name> --query '{TotalNodes:nodeCounts}'`

SHOULD:
- List specific nodes to find problematic signals: `aws iotfleetwise list-signal-catalog-nodes --name <name> --max-results 50`
- Check if the signal exists in the catalog before referencing it in manifests
- Verify signal data types and paths are correctly defined

MAY:
- Review CloudTrail for signal catalog modification events
- Check dependent model manifests: `aws iotfleetwise list-model-manifests --signal-catalog-arn <arn>`
- Validate signal hierarchy (branches and signals follow correct path structure)

## Phase 2 — Remediate

MUST:
- Create signal catalog if none exists: `aws iotfleetwise create-signal-catalog --name <name> --nodes <signal-definitions>`
- Add missing signals: `aws iotfleetwise update-signal-catalog --name <name> --nodes-to-add <nodes>`
- Fix signal definitions (data type, path, description): update individual nodes

SHOULD:
- Follow VSS (Vehicle Signal Specification) naming conventions for signal paths
- Organize signals in a hierarchical branch structure (e.g., Vehicle.Chassis.Brake.PedalPosition)
- Validate signal data types match expected vehicle data formats

MAY:
- Import signals from a VSS-compatible JSON file
- Document signal catalog schema for cross-team reference
- Set up change management process for signal catalog modifications

## Safety Ratings

```
safety_ratings:
  - "list-signal-catalogs: GREEN — read-only enumeration"
  - "get-signal-catalog: GREEN — read-only inspection"
  - "update-signal-catalog --nodes-to-add: YELLOW — adds signals, may affect dependent manifests"
  - "update-signal-catalog --nodes-to-remove: RED — removing signals breaks dependent manifests"
  - "delete-signal-catalog: RED — destroys all FleetWise configuration"
```

## Escalation Conditions

- Signal catalog corruption affecting all FleetWise operations
- Signal removal needed but dependent manifests and campaigns are active
- Signal catalog quota limits reached (maximum nodes per catalog)
- Cross-team signal naming conflicts requiring governance decision

## Data Sensitivity

- `aws iotfleetwise get-signal-catalog`: LOW — returns signal definitions
- `aws iotfleetwise list-signal-catalog-nodes`: LOW — returns signal hierarchy
- Signal definitions: MEDIUM — may reveal proprietary vehicle data architecture

## Prohibited Actions

- NEVER delete the signal catalog — it destroys all FleetWise configuration in the account/region
- NEVER remove signals that are referenced by active model manifests
- NEVER create a second signal catalog — only one is allowed per account per region
- NEVER modify signal data types for signals used in active campaigns without testing

## Phase 3 — Rollback

1. If signals were incorrectly added, remove them: `aws iotfleetwise update-signal-catalog --name <name> --nodes-to-remove <fully-qualified-names>`
2. If signals were incorrectly removed, re-add them with original definitions
3. If signal definitions were modified, restore original data types and paths
4. Verify all dependent model manifests still validate correctly
5. Check active campaigns are not affected by signal catalog changes

## Common Issues

- symptoms: "CreateSignalCatalog returns ResourceAlreadyExistsException"
  diagnosis: "A signal catalog already exists in this account/region. Only one is allowed."
  resolution: "Use the existing catalog. Add signals to it: aws iotfleetwise update-signal-catalog --name <existing-name> --nodes-to-add <nodes>"

- symptoms: "Model manifest references signal path that does not exist in catalog"
  diagnosis: "Signal was not added to the catalog, or the path in the manifest does not match the catalog path exactly."
  resolution: "Add the signal to the catalog or fix the path reference in the model manifest to match exactly."

## Output Format

```yaml
root_cause: "signal_catalog — <specific_cause>"
evidence:
  - type: catalog_error
    content: "<error details>"
severity: CRITICAL
mitigation:
  immediate: "Fix signal catalog configuration"
  long_term: "Implement signal catalog governance and change management"
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
