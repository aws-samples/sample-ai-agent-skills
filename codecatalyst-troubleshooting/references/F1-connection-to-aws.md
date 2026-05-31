---
title: "F1 — Connection to AWS"
description: "Diagnose AWS account connection issues"
status: active
severity: HIGH
triggers: ["AWS connection failed", "cannot connect AWS account", "connection error", "account link broken"]
owner: devops-agent
objective: "Resolve AWS account connection and integration issues"
context: "AWS connections link CodeCatalyst spaces to AWS accounts via IAM roles with specific trust policies."
---

## Phase 1 — Triage

MUST:
- Verify space exists and is active: `aws codecatalyst get-space --name <space>`
- Check AWS connection status in CodeCatalyst console
- Verify IAM role exists in target account: `aws iam get-role --role-name <codecatalyst-connection-role>`

SHOULD:
- Check IAM role trust policy: `aws iam get-role --role-name <role> --query 'Role.AssumeRolePolicyDocument'`
- Verify the trust policy includes codecatalyst.amazonaws.com
- Review CloudTrail for AssumeRole failures: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=AssumeRole --max-results 20`

MAY:
- Check if SCP or permission boundary blocks the role assumption
- Verify the AWS account is not suspended

## Phase 2 — Remediate

MUST:
- Create or fix IAM role with correct trust policy for CodeCatalyst
- Ensure trust policy includes the correct CodeCatalyst service principal
- Verify IAM role has necessary permissions for intended operations

SHOULD:
- Use least-privilege permissions for the connection role
- Add condition keys to the trust policy for additional security
- Document the connection configuration

MAY:
- Set up multiple connections for different environments (dev, staging, prod)
- Configure connection-level permissions restrictions

## Safety Ratings

```
safety_ratings:
  - "codecatalyst get-space: GREEN — read-only space details"
  - "iam get-role: GREEN — read-only role details"
  - "iam create-role: YELLOW — creates IAM role with trust relationship"
  - "iam update-assume-role-policy: YELLOW — modifies trust relationship"
```

## Escalation Conditions

- AWS connection cannot be established after IAM role fix
- SCP blocks CodeCatalyst service principal from assuming roles
- Multiple AWS accounts need simultaneous connection fixes
- Connection failure blocks production deployments

## Data Sensitivity

- `aws codecatalyst get-space`: LOW — returns space metadata
- `aws iam get-role`: MEDIUM — reveals role configuration and trust relationships
- IAM role policies: MEDIUM — reveals permissions granted to CodeCatalyst
- CloudTrail AssumeRole events: MEDIUM — reveals cross-service access patterns

## Prohibited Actions

- NEVER grant AdministratorAccess to CodeCatalyst connection roles
- NEVER use wildcard (*) in trust policy conditions
- NEVER share IAM role ARNs publicly
- NEVER create connection roles without organizational security review

## Phase 3 — Rollback

1. If IAM role was modified, restore original trust policy and permissions
2. If a new role was created, delete it if the connection is not needed
3. Verify existing workflows using the connection still function
4. Confirm no unauthorized access occurred through the modified role
5. Document connection changes and notify the security team

## Common Issues

- symptoms: "AWS connection shows 'failed' status"
  diagnosis: "IAM role does not exist or trust policy is incorrect."
  resolution: "Create IAM role with trust policy allowing codecatalyst.amazonaws.com. Include external ID if required."

- symptoms: "Connection works but workflows fail with AccessDenied"
  diagnosis: "IAM role exists but lacks permissions for workflow actions."
  resolution: "Add required permissions to the IAM role policy for the specific AWS services used in workflows."

## Output Format

```yaml
root_cause: "aws_connection — <specific_cause>"
evidence:
  - type: connection_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix AWS connection"
  long_term: "Document connection setup and implement monitoring"
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
