---
title: "A2 — Configured Model Errors"
description: "Diagnose configured model algorithm and configured audience model errors"
status: active
severity: HIGH
triggers: ["configured model error", "model configuration failed", "configured audience model"]
owner: devops-agent
objective: "Resolve configured model errors and ensure proper model-collaboration association"
context: "Configured models bridge trained models to collaborations. Configured audience models define how audience generation operates within a collaboration context."
---

## Phase 1 — Triage

MUST:
- List configured model algorithms: `aws cleanroomsml list-configured-model-algorithms --query 'configuredModelAlgorithms[*].{Name:name,Arn:configuredModelAlgorithmArn}'`
- Get configured audience model details: `aws cleanroomsml get-configured-audience-model --configured-audience-model-arn <arn>`
- Check audience model status: `aws cleanroomsml get-audience-model --audience-model-arn <arn>`

SHOULD:
- Verify the underlying audience model is in ACTIVE status
- Check collaboration association status: `aws cleanroomsml list-configured-audience-model-associations`

MAY:
- Review CloudTrail for configuration change events
- Check if privacy budget is configured for the association

## Phase 2 — Remediate

MUST:
- Ensure the underlying audience model is ACTIVE before configuring
- Verify output configuration S3 bucket exists and has proper permissions
- Fix collaboration association if missing or misconfigured

SHOULD:
- Validate that all collaboration members have accepted the ML configuration
- Ensure KMS key is accessible by all required principals

MAY:
- Recreate the configured audience model with corrected parameters

## Common Issues

- symptoms: "Cannot create configured audience model association"
  diagnosis: "Collaboration does not have ML configured or member has not accepted."
  resolution: "Verify collaboration ML configuration. Ensure all required members have accepted ML terms."

- symptoms: "Configured audience model shows incorrect output settings"
  diagnosis: "Output S3 bucket or KMS key misconfigured during creation."
  resolution: "Delete and recreate the configured audience model with correct output settings. Cannot update in place."

## Safety Ratings

```
safety_ratings:
  - "get-configured-audience-model: GREEN — read-only detail retrieval"
  - "list-configured-model-algorithms: GREEN — read-only enumeration"
  - "create-configured-audience-model: YELLOW — creates configuration resource"
  - "delete-configured-audience-model: RED — removes configuration, breaks audience generation"
```

## Escalation Conditions

- Configured model association fails with internal errors after verifying all permissions
- Collaboration members report conflicting ML configuration states
- Configured model references a deleted or corrupted audience model
- Cross-account collaboration ML setup requires coordination with external parties

## Data Sensitivity

- `aws cleanroomsml get-configured-audience-model`: MEDIUM — reveals model configuration and output destinations
- `aws cleanroomsml list-configured-model-algorithms`: LOW — returns metadata only
- `aws cleanrooms get-collaboration`: MEDIUM — reveals collaboration member details
- `aws cloudtrail lookup-events`: MEDIUM — may reveal IAM principal identities

## Prohibited Actions

- NEVER delete a configured audience model that has active audience generation jobs
- NEVER modify collaboration ML settings without agreement from all collaboration members
- NEVER expose configured model ARNs to parties outside the collaboration
- NEVER change output S3 destinations without verifying data governance requirements

## Phase 3 — Rollback

1. If a configured audience model was created with wrong settings, delete it: `aws cleanroomsml delete-configured-audience-model --configured-audience-model-arn <arn>`
2. If collaboration association was broken, recreate it with correct parameters
3. Verify audience generation jobs are not in progress before any rollback
4. Notify collaboration members of configuration changes
5. Document the error and resolution for audit trail

## Output Format

```yaml
root_cause: "configured_model_error — <specific_cause>"
evidence:
  - type: configuration_error
    content: "<error details from API response>"
severity: HIGH
mitigation:
  immediate: "Fix configuration and reassociate with collaboration"
  long_term: "Use IaC to manage configured model lifecycle"
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
