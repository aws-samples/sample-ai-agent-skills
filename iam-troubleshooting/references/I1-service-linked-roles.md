---
title: "I1 — Service-Linked Roles"
description: "Diagnose service-linked role creation, deletion, and permission issues"
status: active
severity: MEDIUM
triggers:
  - "service-linked role"
  - "SLR"
  - "AWSServiceRoleFor"
  - "cannot delete service role"
  - "service role creation"
owner: devops-agent
objective: "Resolve service-linked role issues including creation failures, deletion blocks, and permission understanding"
context: "Service-linked roles (SLRs) are IAM roles managed by AWS services. They have predefined permissions that cannot be modified. SLRs are created automatically by services or manually. Deletion requires the service to release the role first. SCPs do not apply to SLRs."
---

## Phase 1 — Triage

MUST:
- Identify the service-linked role: `aws iam get-role --role-name AWSServiceRoleFor<ServiceName>`
- Check if the role exists: `aws iam list-roles --query "Roles[?starts_with(Path, '/aws-service-role/')]"`
- Check the role's permissions: `aws iam list-attached-role-policies --role-name <slr-name>` and `aws iam list-role-policies --role-name <slr-name>`
- Verify the service that owns the role

SHOULD:
- Check if the principal has permission to create the SLR: `iam:CreateServiceLinkedRole`
- Check if the service has resources that depend on the SLR (blocking deletion)
- Review CloudTrail for SLR creation/deletion events

MAY:
- Check if the SLR was created automatically by the service or manually
- Verify the SLR's trust policy (should trust the specific service)

## Phase 2 — Remediate

MUST:
- To create an SLR: `aws iam create-service-linked-role --aws-service-name <service>.amazonaws.com`
- To delete an SLR: first remove all service resources that depend on it, then: `aws iam delete-service-linked-role --role-name <slr-name>`
- Check deletion status: `aws iam get-service-linked-role-deletion-status --deletion-task-id <task-id>`

SHOULD:
- Grant `iam:CreateServiceLinkedRole` to principals that need to enable AWS services
- Do not attempt to modify SLR permissions — they are managed by AWS
- Clean up service resources before attempting SLR deletion

MAY:
- Use SCPs to prevent SLR creation for services you don't want enabled (note: SCPs don't restrict SLR actions, but can restrict the service actions that trigger SLR creation)
- Document which SLRs exist and their purpose

## Common Issues

- symptoms: "Cannot create service-linked role — AccessDenied"
  diagnosis: "The principal lacks iam:CreateServiceLinkedRole permission."
  resolution: "Add `iam:CreateServiceLinkedRole` with a condition on `iam:AWSServiceName` for the specific service."

- symptoms: "Cannot delete service-linked role — 'role is being used'"
  diagnosis: "The service still has resources that depend on the SLR."
  resolution: "Delete or disable the service resources first, then retry SLR deletion."

- symptoms: "Service-linked role has more permissions than expected"
  diagnosis: "SLR permissions are defined by AWS and cannot be modified. They include what the service needs."
  resolution: "This is by design. Use SCPs or IAM policies on the calling principal to restrict what the service can do."

## Output Format

```yaml
root_cause: "service_linked_role — <specific_cause>"
evidence:
  - type: slr_details
    content: "<role name, service, creation date>"
  - type: slr_permissions
    content: "<attached policies>"
  - type: dependent_resources
    content: "<service resources using the SLR>"
severity: MEDIUM
mitigation:
  immediate: "Create, delete, or accept SLR as needed"
  long_term: "Document SLRs and implement service governance"
```


## Safety Ratings

safety_ratings:
  - "Service-linked role management (SLRs are AWS-managed with predefined permissions): GREEN — SLR permissions cannot be modified; creation and deletion are controlled by AWS service lifecycle"

## Escalation Conditions

escalation_conditions:
  - "Escalate to a human if SLR deletion is blocked by dependent service resources that cannot be easily identified"
  - "Escalate if the SLR permissions are broader than expected and the service cannot be restricted"
  - "Escalate if SLR creation is needed for a service that will have broad access to account resources"

## Rollback

If remediation makes things worse:
1. Document the change that was made
2. Recreate the deleted SLR using `aws iam create-service-linked-role --aws-service-name <service>.amazonaws.com`; note that SLR permissions are AWS-managed and will be restored to the standard configuration
3. Verify access is restored to pre-change state

## Data Sensitivity

- `aws iam get-role` for SLRs returns the role configuration and trust policy — classification: LOW
- `aws iam list-attached-role-policies` for SLRs reveals AWS-managed policy ARNs — classification: LOW
- Service resource inventory (needed to unblock SLR deletion) may reveal infrastructure details — classification: MEDIUM

## Prohibited Actions

- NEVER suggest `"Action": "*"` or `"Resource": "*"` in any policy fix
- NEVER suggest granting AdministratorAccess to resolve permission issues
- NEVER suggest removing permission boundaries to fix access denied
- Always identify the SPECIFIC denied action from CloudTrail before suggesting a fix
- Always scope fixes to the minimum required permissions (least privilege)

## Cleanup

- After deleting service resources to unblock SLR deletion, verify the SLR deletion completed using `aws iam get-service-linked-role-deletion-status --deletion-task-id <task-id>`

## Data Sensitivity

data_sensitivity:
  - command: "get-account-authorization-details"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-policy-version"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-role"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest Action: * or Resource: * in any policy"
  - "NEVER suggest removing permission boundaries"
  - "NEVER suggest AdministratorAccess for troubleshooting"

## Phase 3 � Rollback

- Document the current state before making any changes
- If remediation worsens the issue, revert to the documented pre-change state
- Remove any temporary resources created during investigation
- Verify the system returns to its previous functional state after rollback
