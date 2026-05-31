---
title: "H2 — Cross-Account Resource Sharing"
description: "Diagnose cross-account resource sharing via RAM and resource policies"
status: active
severity: MEDIUM
triggers:
  - "resource sharing"
  - "RAM"
  - "Resource Access Manager"
  - "share resource"
  - "cross-account resource"
owner: devops-agent
objective: "Configure and troubleshoot cross-account resource sharing using RAM or resource policies"
context: "Cross-account resource access can be granted via: (1) resource-based policies (S3, KMS, SQS, SNS, Lambda, etc.), (2) AWS RAM for supported resources (subnets, Transit Gateway, Route 53 Resolver, etc.), or (3) cross-account roles. The best approach depends on the resource type and use case."
---

## Phase 1 — Triage

MUST:
- Identify the resource type and determine the sharing mechanism
- For RAM: `aws ram get-resource-shares --resource-owner SELF` or `aws ram get-resource-shares --resource-owner OTHER-ACCOUNTS`
- For resource policies: check the specific service's resource policy
- Verify the target account/principal has accepted the RAM share (if applicable)

SHOULD:
- Check if the resource type supports resource-based policies
- Check if the resource type is supported by RAM: `aws ram list-resource-types`
- Verify Organizations integration if sharing within an org: `aws ram enable-sharing-with-aws-organization`

MAY:
- Check if the resource is in a VPC that requires additional networking setup
- Review AWS Config for resource sharing compliance rules

## Phase 2 — Remediate

MUST:
- For RAM sharing:
  - Create share: `aws ram create-resource-share --name <name> --resource-arns <resource-arn> --principals <account-id>`
  - Accept share (in target account): `aws ram accept-resource-share-invitation --resource-share-invitation-arn <arn>`
- For resource policies: add cross-account Allow statement with the target account principal
- Ensure the target account's IAM policies also allow the action (cross-account requires both sides)

SHOULD:
- Use Organizations sharing to auto-accept shares within the org
- Apply least-privilege in both the resource policy and the target account's IAM policy
- Use conditions (aws:PrincipalOrgID, aws:SourceAccount) for security

MAY:
- Use AWS PrivateLink for cross-account service access without public internet
- Implement tagging standards for shared resources

## Common Issues

- symptoms: "RAM share is pending — target account hasn't accepted"
  diagnosis: "RAM shares require acceptance unless Organizations sharing is enabled."
  resolution: "Accept the share in the target account or enable Organizations sharing."

- symptoms: "Resource policy allows cross-account but access is denied"
  diagnosis: "The target account's IAM policy doesn't have the corresponding permission."
  resolution: "Add the permission to the target account principal's IAM policy."

- symptoms: "Cannot share subnet via RAM — 'not supported'"
  diagnosis: "RAM sharing may not be enabled for the Organization, or the resource type isn't supported in the region."
  resolution: "Enable RAM sharing with the Organization: `aws ram enable-sharing-with-aws-organization`"

## Output Format

```yaml
root_cause: "resource_sharing — <mechanism>: <specific_cause>"
evidence:
  - type: sharing_mechanism
    content: "RAM | resource_policy | cross_account_role"
  - type: share_status
    content: "<active|pending|failed>"
  - type: permissions
    content: "<resource policy and IAM policy status>"
severity: MEDIUM
mitigation:
  immediate: "Fix sharing configuration or accept pending shares"
  long_term: "Standardize on RAM with Organizations integration"
```


## Safety Ratings

safety_ratings:
  - "Cross-account resource sharing (RAM shares and resource policy modifications): YELLOW — Resource sharing changes can expose resources to other accounts; misconfiguration can grant unintended access"

## Escalation Conditions

escalation_conditions:
  - "Escalate to a human if sharing resources with accounts outside the organization (external sharing)"
  - "Escalate if the resource being shared contains sensitive data (S3 buckets, KMS keys, databases)"
  - "Escalate if enabling RAM organization-wide sharing"

## Rollback

If remediation makes things worse:
1. Document the change that was made
2. Delete the RAM resource share using `aws ram delete-resource-share --resource-share-arn <arn>`, or restore the original resource policy (e.g., `aws s3api put-bucket-policy --bucket <bucket> --policy file://original-policy.json`)
3. Verify access is restored to pre-change state

## Data Sensitivity

- `aws ram get-resource-shares` reveals all resource sharing relationships including target accounts — classification: HIGH
- Resource policies (S3, KMS, SQS, etc.) reveal cross-account access grants — classification: HIGH
- `aws ram list-principals` reveals which accounts have access to shared resources — classification: MEDIUM

## Prohibited Actions

- NEVER suggest `"Action": "*"` or `"Resource": "*"` in any policy fix
- NEVER suggest granting AdministratorAccess to resolve permission issues
- NEVER suggest removing permission boundaries to fix access denied
- Always identify the SPECIFIC denied action from CloudTrail before suggesting a fix
- Always scope fixes to the minimum required permissions (least privilege)

## Cleanup

- Delete RAM resource shares that are no longer needed: `aws ram delete-resource-share --resource-share-arn <arn>`
- Remove cross-account principals from resource policies when sharing is no longer required

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
