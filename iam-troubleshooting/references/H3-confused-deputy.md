---
title: "H3 — Confused Deputy Prevention"
description: "Prevent confused deputy attacks using ExternalId and source conditions"
status: active
severity: MEDIUM
triggers:
  - "confused deputy"
  - "ExternalId"
  - "aws:SourceAccount"
  - "aws:SourceArn"
  - "third-party access"
owner: devops-agent
objective: "Implement confused deputy prevention for cross-account and service access patterns"
context: "The confused deputy problem occurs when a trusted entity (service or third-party account) is tricked into acting on behalf of an unauthorized party. Prevention uses: sts:ExternalId for third-party role assumption, aws:SourceAccount and aws:SourceArn for service-to-service access."
---

## Phase 1 — Triage

MUST:
- Check if the role is used by a third party or AWS service: `aws iam get-role --role-name <role> --query 'Role.AssumeRolePolicyDocument'`
- Verify ExternalId condition exists for third-party roles
- Verify aws:SourceAccount/aws:SourceArn conditions exist for service roles
- Check if the trust policy uses wildcard (*) in Principal (high risk)

SHOULD:
- Review all cross-account roles for confused deputy protections
- Check if the ExternalId is unique per customer (not shared across customers)
- Verify aws:SourceArn is as specific as possible (not just account-level)

MAY:
- Audit all roles with cross-account trust using IAM Access Analyzer
- Check for roles trusting third-party accounts without ExternalId

## Phase 2 — Remediate

MUST:
- For third-party roles, add ExternalId condition:
  ```json
  {
    "Effect": "Allow",
    "Principal": {"AWS": "<third-party-account>"},
    "Action": "sts:AssumeRole",
    "Condition": {"StringEquals": {"sts:ExternalId": "<unique-id>"}}
  }
  ```
- For service roles, add source conditions:
  ```json
  {
    "Condition": {
      "StringEquals": {"aws:SourceAccount": "<your-account-id>"},
      "ArnLike": {"aws:SourceArn": "arn:aws:service:region:<account>:resource/*"}
    }
  }
  ```

SHOULD:
- Generate unique ExternalId per third-party relationship (UUID recommended)
- Use both aws:SourceAccount AND aws:SourceArn for defense in depth
- Document the ExternalId and share it securely with the third party

MAY:
- Implement automated scanning for roles missing confused deputy protections
- Use AWS Config custom rules to enforce ExternalId on cross-account roles

## Common Issues

- symptoms: "Third-party vendor says ExternalId doesn't match"
  diagnosis: "The ExternalId in the trust policy doesn't match what the vendor passes in AssumeRole."
  resolution: "Coordinate with the vendor to ensure the ExternalId matches exactly. It's case-sensitive."

- symptoms: "Service role works without source conditions but security audit flags it"
  diagnosis: "The role is vulnerable to confused deputy attacks without source conditions."
  resolution: "Add aws:SourceAccount and aws:SourceArn conditions to the trust policy."

- symptoms: "Adding ExternalId breaks existing third-party integration"
  diagnosis: "The third party must pass the ExternalId in their AssumeRole call."
  resolution: "Coordinate with the third party to update their configuration with the ExternalId."

## Output Format

```yaml
root_cause: "confused_deputy — <specific_vulnerability>"
evidence:
  - type: trust_policy
    content: "<current trust policy>"
  - type: protection_status
    content: "ExternalId: <present|missing>, SourceConditions: <present|missing>"
severity: MEDIUM
mitigation:
  immediate: "Add ExternalId or source conditions to trust policy"
  long_term: "Audit all cross-account roles and implement automated compliance checks"
```


## Safety Ratings

safety_ratings:
  - "Confused deputy prevention (ExternalId and source condition changes to trust policies): YELLOW — Adding or modifying ExternalId conditions can break existing third-party integrations if not coordinated"

## Escalation Conditions

escalation_conditions:
  - "Escalate to a human if the trust policy change affects an active third-party integration"
  - "Escalate if the role is used by multiple third parties with different ExternalIds"
  - "Escalate if removing or changing an ExternalId on a production role"

## Rollback

If remediation makes things worse:
1. Document the change that was made
2. Restore the original ExternalId condition by reverting the trust policy using `aws iam update-assume-role-policy --role-name <role> --policy-document file://original-trust-policy.json`
3. Verify access is restored to pre-change state

## Data Sensitivity

- `aws iam get-role` returns trust policy documents including ExternalId values and trusted third-party account IDs — classification: HIGH
- ExternalId values are shared secrets between you and the third party — classification: HIGH
- IAM Access Analyzer findings may reveal roles with cross-account trust lacking confused deputy protections — classification: MEDIUM

## Prohibited Actions

- NEVER suggest `"Action": "*"` or `"Resource": "*"` in any policy fix
- NEVER suggest granting AdministratorAccess to resolve permission issues
- NEVER suggest removing permission boundaries to fix access denied
- Always identify the SPECIFIC denied action from CloudTrail before suggesting a fix
- Always scope fixes to the minimum required permissions (least privilege)

## Cleanup

- No temporary resources are created during confused deputy prevention configuration

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
