---
title: "E2 — IAM Permissions"
description: "Diagnose Amazon Lookout for Equipment IAM permission issues"
status: active
severity: HIGH
triggers: ["AccessDeniedException", "not authorized", "IAM permission error", "assume role failed"]
owner: devops-agent
objective: "Resolve IAM permission issues for Lookout for Equipment API operations"
context: "Lookout for Equipment requires IAM permissions for API calls and a service role for data access. Common issues include missing API permissions and incorrect trust policies."
---

## Phase 1 — Triage

MUST:
- Check caller identity: `aws sts get-caller-identity`
- Verify IAM policies attached to the caller: `aws iam list-attached-user-policies --user-name <name>` or `aws iam list-attached-role-policies --role-name <name>`
- Simulate the failing action: `aws iam simulate-principal-policy --policy-source-arn <arn> --action-names lookoutequipment:<action>`
- Check for SCPs that may deny access: `aws organizations list-policies --filter SERVICE_CONTROL_POLICY`

SHOULD:
- Check if the caller is using the correct AWS profile or role
- Verify no permission boundaries restrict lookoutequipment actions
- Check CloudTrail for the specific AccessDenied event details

MAY:
- Review IAM Access Advisor for last-used lookoutequipment permissions
- Check if the region has lookoutequipment service endpoint available

## Phase 2 — Remediate

MUST:
- Grant lookoutequipment:* or specific action permissions to the calling principal
- For service roles: ensure the trust policy includes lookoutequipment.amazonaws.com
- For cross-account access: set up proper IAM role assumption chain
- Verify no SCPs at the OU or account level deny lookoutequipment actions

SHOULD:
- Use least-privilege permissions — grant only the specific lookoutequipment actions needed
- Create a dedicated service role for Lookout for Equipment rather than reusing existing roles
- Tag the IAM role for cost allocation and access tracking

MAY:
- Implement IAM permission boundaries for additional security
- Set up AWS Config rules to monitor IAM policy compliance

## Common Issues

- symptoms: "AccessDeniedException when calling CreateDataset"
  diagnosis: "The calling principal lacks lookoutequipment:CreateDataset permission."
  resolution: "Attach a policy with lookoutequipment:CreateDataset to the user or role. For full access: lookoutequipment:*."

- symptoms: "Unable to pass role to Lookout for Equipment"
  diagnosis: "The caller lacks iam:PassRole permission for the service role."
  resolution: "Grant iam:PassRole with a condition restricting to lookoutequipment.amazonaws.com: add iam:PassRole with condition iam:PassedToService equals lookoutequipment.amazonaws.com."

- symptoms: "Service role cannot access S3 despite correct S3 permissions"
  diagnosis: "The role trust policy does not allow lookoutequipment.amazonaws.com to assume it."
  resolution: "Update the trust policy to include: {\"Effect\": \"Allow\", \"Principal\": {\"Service\": \"lookoutequipment.amazonaws.com\"}, \"Action\": \"sts:AssumeRole\"}."

## Safety Ratings

```
safety_ratings:
  - "sts get-caller-identity: GREEN — read-only identity check"
  - "iam list-attached-*-policies: GREEN — read-only policy enumeration"
  - "iam simulate-principal-policy: GREEN — read-only permission simulation"
  - "Attaching IAM policies: YELLOW — grants new permissions"
  - "Modifying trust policies: RED — changes who can assume the role"
```

## Escalation Conditions

- SCPs at the organization level deny lookoutequipment access and require org admin changes
- Permission boundaries set by security team prevent granting required permissions
- Cross-account role assumption chain is complex and involves multiple accounts
- IAM changes require change management approval in regulated environments

## Data Sensitivity

- `aws sts get-caller-identity`: LOW — returns account ID and principal ARN
- `aws iam list-attached-*-policies`: MEDIUM — reveals IAM policy attachments
- `aws iam simulate-principal-policy`: LOW — tests permissions without changes
- IAM policy documents: MEDIUM — reveal access control configuration
- SCP policies: HIGH — reveal organization-wide security controls

## Prohibited Actions

- NEVER grant AdministratorAccess or lookoutequipment:* in production without justification
- NEVER modify SCPs without organization administrator approval
- NEVER remove permission boundaries set by the security team
- NEVER share IAM policy documents or role ARNs externally without redacting account IDs

## Phase 3 — Rollback

1. If IAM policy was incorrectly modified, detach it: `aws iam detach-role-policy --role-name <name> --policy-arn <arn>`
2. If trust policy was changed, restore the previous version from CloudTrail records
3. If a new role was created in error, delete it after removing all policies: `aws iam delete-role --role-name <name>`
4. Verify all Lookout for Equipment operations work with the corrected permissions
5. Document the minimum required permissions for the operations runbook

## Output Format

```yaml
root_cause: "iam_permissions — <specific_cause>"
evidence:
  - type: permission_error
    content: "<AccessDenied details from CloudTrail>"
severity: HIGH
mitigation:
  immediate: "Grant required IAM permissions"
  long_term: "Implement least-privilege IAM policies with automated compliance checks"
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
