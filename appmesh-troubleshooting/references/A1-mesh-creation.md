---
title: "A1 — App Mesh Creation Issues"
description: "Diagnose mesh creation and initial setup failures"
status: active
severity: HIGH
triggers:
  - "mesh creation failed"
  - "cannot create mesh"
  - "mesh error"
owner: devops-agent
objective: "Identify why an App Mesh fails to create and resolve the issue"
context: "Mesh creation is the first step in App Mesh setup. Failures are typically caused by IAM permission issues, naming conflicts, or account-level limits."
---

## Phase 1 — Triage

MUST:
- List existing meshes: `aws appmesh list-meshes`
- Check if mesh exists: `aws appmesh describe-mesh --mesh-name <mesh-name>`
- Check CloudTrail for creation events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateMesh --max-results 10`
- Verify IAM permissions: `aws iam simulate-principal-policy --policy-source-arn <caller-arn> --action-names appmesh:CreateMesh`

SHOULD:
- Check for naming conflicts with existing meshes
- Verify service quotas for App Mesh: `aws service-quotas get-service-quota --service-code appmesh --quota-code L-AC861A42`
- Check if mesh egress filter is set correctly (ALLOW_ALL or DROP_ALL)

MAY:
- Check if cross-account sharing is needed via RAM
- Verify the mesh spec configuration (egress filter, service discovery type)

## Phase 2 — Remediate

MUST:
- Grant `appmesh:CreateMesh` permission to the calling identity
- Use a unique mesh name that doesn't conflict with existing meshes
- Configure egress filter based on security requirements

SHOULD:
- Use ALLOW_ALL egress filter initially for easier debugging, then restrict
- Tag meshes for resource management
- Document mesh ownership and purpose

MAY:
- Share mesh via RAM for cross-account use
- Set up mesh with IaC (CloudFormation, Terraform, or EKS CRDs)

## Common Issues

- symptoms: "AccessDeniedException when creating mesh"
  diagnosis: "Calling identity lacks appmesh:CreateMesh permission."
  resolution: "Add App Mesh permissions to the IAM policy."

- symptoms: "Mesh name conflict"
  diagnosis: "A mesh with the same name already exists in the account/region."
  resolution: "Use a different name or delete the existing mesh if unused."

- symptoms: "Mesh created but services cannot communicate"
  diagnosis: "Egress filter set to DROP_ALL blocking traffic to services not in the mesh."
  resolution: "Set egress filter to ALLOW_ALL or add all services to the mesh."

## Output Format

```yaml
root_cause: "mesh_creation_issue — <specific_cause>"
evidence:
  - type: mesh_status
    content: "<mesh details or error>"
  - type: iam_permissions
    content: "<caller permissions>"
  - type: cloudtrail
    content: "<creation event details>"
severity: HIGH
mitigation:
  immediate: "Fix permissions or naming and retry creation"
  long_term: "Use IaC for mesh provisioning with validated configurations"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Grant appmesh:CreateMesh permission | YELLOW | IAM policy change |
| Use a unique mesh name | GREEN | Non-destructive naming |
| Configure egress filter | YELLOW | Affects traffic routing behavior |
| Share mesh via RAM | YELLOW | Cross-account access change |
| Set up mesh with IaC | GREEN | Infrastructure as code |

## Escalation Conditions

- Fix requires modifying Envoy proxy config in production
- Fix requires changing routing rules

## Data Sensitivity

- **Mesh configuration**: Egress filter, service discovery type — INTERNAL
- **IAM permissions**: Caller identity and permissions — INTERNAL
- **CloudTrail events**: Mesh creation audit trail — INTERNAL

## Prohibited Actions

- NEVER suggest removing mTLS to fix connectivity
- NEVER modify mesh egress filter in production without testing in staging first
- NEVER delete a mesh that has active virtual nodes or services

## Phase 3 — Rollback

1. Delete the newly created mesh if it caused issues: `aws appmesh delete-mesh --mesh-name <mesh-name>`
2. Revert IAM policy changes if permissions were modified
3. Remove RAM share if cross-account sharing was configured
4. Verify no virtual nodes or services were affected
5. Restore previous mesh configuration if an update was made

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
