---
title: "A2 — Vehicle Model Errors"
description: "Diagnose model manifest creation, activation, and vehicle association failures"
status: active
severity: HIGH
triggers: ["model manifest error", "vehicle model failed", "manifest not active", "cannot create vehicle from model"]
owner: devops-agent
objective: "Resolve vehicle model manifest issues and ensure vehicles can be created"
context: "Model manifests define the signal structure for a vehicle type. They reference signals from the signal catalog. Manifests must be ACTIVE before vehicles can use them."
---

## Phase 1 — Triage

MUST:
- List model manifests: `aws iotfleetwise list-model-manifests --query 'summaries[*].{Name:name,Status:status,Arn:arn}'`
- Describe the failing manifest: `aws iotfleetwise get-model-manifest --name <name>`
- Check manifest status (must be ACTIVE): verify status field in response

SHOULD:
- List signals in the manifest: `aws iotfleetwise list-model-manifest-nodes --name <name>`
- Verify referenced signals exist in signal catalog: `aws iotfleetwise get-signal-catalog --name <catalog-name>`
- Check if decoder manifest exists for this model: `aws iotfleetwise list-decoder-manifests --model-manifest-arn <arn>`

MAY:
- Review CloudTrail for manifest creation/update events
- Check for validation errors in manifest node definitions
- Compare manifest signals with signal catalog entries

## Phase 2 — Remediate

MUST:
- Ensure all signals referenced in model manifest exist in the signal catalog
- Activate the manifest if in DRAFT: `aws iotfleetwise update-model-manifest --name <name> --status ACTIVE`
- Create a decoder manifest for the model if missing

SHOULD:
- Validate signal paths and data types match signal catalog definitions
- Ensure manifest has at least one signal node defined
- Verify signal catalog ARN in manifest matches the account's catalog

MAY:
- Create manifest via CloudFormation for version control
- Document signal schema for the vehicle type
- Set up validation pipeline for manifest changes

## Safety Ratings

```
safety_ratings:
  - "list-model-manifests: GREEN — read-only enumeration"
  - "get-model-manifest: GREEN — read-only inspection"
  - "update-model-manifest --status ACTIVE: YELLOW — activates manifest, enables vehicle creation"
  - "delete-model-manifest: RED — removes manifest, breaks dependent vehicles"
```

## Escalation Conditions

- Model manifest activation fails with validation errors that cannot be resolved
- Signal catalog changes break existing model manifests
- Large number of vehicles depend on a manifest that needs modification
- Manifest schema change required for production vehicle fleet

## Data Sensitivity

- `aws iotfleetwise get-model-manifest`: LOW — returns manifest metadata and signal structure
- `aws iotfleetwise list-model-manifest-nodes`: LOW — returns signal definitions
- `aws iotfleetwise get-signal-catalog`: LOW — returns signal catalog structure

## Prohibited Actions

- NEVER delete a model manifest that has vehicles associated with it
- NEVER modify signal references in a manifest used by production vehicles without testing
- NEVER activate a manifest without verifying a decoder manifest exists for it
- NEVER assume DRAFT manifests are usable — they must be explicitly activated

## Phase 3 — Rollback

1. If manifest was incorrectly activated, it cannot be reverted to DRAFT — create a new corrected manifest
2. If signals were incorrectly added, update manifest nodes: `aws iotfleetwise update-model-manifest --name <name> --nodes-to-remove <nodes>`
3. If manifest was deleted, recreate with same signal structure
4. Update vehicles to use corrected manifest if needed
5. Verify decoder manifests are updated to match any manifest changes

## Common Issues

- symptoms: "UpdateModelManifest to ACTIVE returns ValidationException"
  diagnosis: "Manifest references signals not found in the signal catalog, or signal data types do not match."
  resolution: "Verify all signal paths in the manifest exist in the signal catalog. Check data types match exactly."

- symptoms: "CreateVehicle fails with 'model manifest not active'"
  diagnosis: "Model manifest is still in DRAFT status."
  resolution: "Activate the manifest: aws iotfleetwise update-model-manifest --name <name> --status ACTIVE"

## Output Format

```yaml
root_cause: "vehicle_model — <specific_cause>"
evidence:
  - type: manifest_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix and activate model manifest"
  long_term: "Implement manifest validation pipeline"
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
