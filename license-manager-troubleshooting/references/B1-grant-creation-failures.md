---
title: "B1 — Grant Creation Failures"
description: "Diagnose license grant creation failures"
status: active
severity: HIGH
triggers: ["grant creation failed", "cannot create grant", "share license failed"]
owner: devops-agent
objective: "Resolve license grant creation issues"
context: "Grants share licenses across accounts. Creation requires proper permissions and valid license ARN. Grants go through a lifecycle before becoming active."
---

## Phase 1 — Triage

MUST:
- Verify the license exists: `aws license-manager list-licenses --query 'Licenses[*].{Arn:LicenseArn,Name:LicenseName,Status:Status}'`
- Check IAM permissions: `aws iam simulate-principal-policy --policy-source-arn <user-arn> --action-names license-manager:CreateGrant`
- Verify the target account ID is valid

SHOULD:
- Check existing grants for the license: `aws license-manager list-distributed-grants --query 'Grants[*].{Arn:GrantArn,Name:GrantName,Status:Status,Grantee:GranteePrincipalArn}'`
- Review CloudTrail for CreateGrant errors

MAY:
- Verify the license allows grant creation (check license terms)
- Check if the target account is in the same organization

## Phase 2 — Remediate

MUST:
- Fix IAM permissions for license-manager:CreateGrant
- Use valid license ARN and target account principal
- Create grant with appropriate allowed operations: `aws license-manager create-grant --grant-name <name> --license-arn <license-arn> --principals <account-arn> --allowed-operations CheckoutLicense CheckInLicense`

SHOULD:
- Notify the recipient to accept the grant
- Monitor grant status until ACTIVE

MAY:
- Set up automated grant creation for new organization members
- Document grant distribution for compliance

## Safety Ratings

```
safety_ratings:
  - "license-manager list-licenses: GREEN — read-only listing"
  - "license-manager list-distributed-grants: GREEN — read-only listing"
  - "license-manager create-grant: YELLOW — shares license access with another account"
  - "iam simulate-principal-policy: GREEN — read-only permission check"
```

## Escalation Conditions

- Grant creation fails with license terms violation
- Target account cannot be reached for grant acceptance
- Grant requires operations not allowed by the license
- Multiple grants needed across many accounts simultaneously

## Data Sensitivity

- `aws license-manager list-licenses`: MEDIUM — reveals license inventory
- Grant details: MEDIUM — reveals cross-account sharing relationships
- Target account ARNs: LOW — account identifiers
- License terms: HIGH — contractual information

## Prohibited Actions

- NEVER create grants with more operations than needed (least privilege)
- NEVER share licenses with accounts outside the organization without approval
- NEVER create grants for licenses you do not own or have distribution rights for
- NEVER bypass grant approval workflows

## Phase 3 — Rollback

1. Delete incorrectly created grants: `aws license-manager delete-grant --grant-arn <arn>`
2. If grant was accepted in error, recipient should return it
3. Verify license consumption is correct after grant changes
4. Document grant creation and deletion for audit trail
5. Notify affected accounts of grant changes

## Common Issues

- symptoms: "CreateGrant returns InvalidParameterValueException"
  diagnosis: "Invalid license ARN, principal ARN, or unsupported allowed operations."
  resolution: "Verify license ARN exists and is active. Check principal ARN format. Use only supported operations."

## Output Format

```yaml
root_cause: "grant_creation — <specific_cause>"
evidence:
  - type: grant_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix grant creation issue"
  long_term: "Automate grant distribution"
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
