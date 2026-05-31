---
title: "I2 — Service Roles and PassRole"
description: "Diagnose service role trust policies and iam:PassRole permission issues"
status: active
severity: MEDIUM
triggers:
  - "service role"
  - "PassRole"
  - "iam:PassRole"
  - "not authorized to pass role"
  - "service trust policy"
owner: devops-agent
objective: "Resolve service role configuration and PassRole permission issues"
context: "Service roles allow AWS services to act on your behalf. Creating or assigning a service role requires iam:PassRole permission. The trust policy must allow the specific service. PassRole is a critical security control — it prevents privilege escalation by controlling who can assign roles to services."
---

## Phase 1 — Triage

MUST:
- Check the service role's trust policy: `aws iam get-role --role-name <role> --query 'Role.AssumeRolePolicyDocument'`
- Verify the trust policy allows the correct service: `"Principal": {"Service": "<service>.amazonaws.com"}`
- Check if the user has iam:PassRole: `aws iam simulate-principal-policy --policy-source-arn <user-arn> --action-names iam:PassRole --resource-arns <role-arn>`
- Verify the PassRole permission includes the specific role ARN

SHOULD:
- Check if the PassRole policy has conditions (e.g., `iam:PassedToService`)
- Verify the role has the permissions the service needs
- Check CloudTrail for PassRole denied events

MAY:
- Review all service roles in the account: `aws iam list-roles --query "Roles[?AssumeRolePolicyDocument.Statement[?Principal.Service]]"`
- Check if the service supports service-linked roles instead

## Phase 2 — Remediate

MUST:
- Grant iam:PassRole to the user for the specific role:
  ```json
  {
    "Effect": "Allow",
    "Action": "iam:PassRole",
    "Resource": "arn:aws:iam::<account>:role/<service-role>",
    "Condition": {
      "StringEquals": {"iam:PassedToService": "<service>.amazonaws.com"}
    }
  }
  ```
- Ensure the service role trust policy allows the service:
  ```json
  {
    "Effect": "Allow",
    "Principal": {"Service": "<service>.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }
  ```

SHOULD:
- Use `iam:PassedToService` condition to restrict which services the role can be passed to
- Scope PassRole to specific role ARNs, not `*`
- Use separate service roles for different services

MAY:
- Implement tagging on service roles for governance
- Use AWS Config to monitor service role configurations

## Common Issues

- symptoms: "Cannot create Lambda function — 'not authorized to perform iam:PassRole'"
  diagnosis: "The user lacks iam:PassRole permission for the Lambda execution role."
  resolution: "Add iam:PassRole for the specific role ARN with iam:PassedToService condition for lambda.amazonaws.com."

- symptoms: "Service role exists but service cannot use it"
  diagnosis: "The trust policy doesn't allow the specific service, or the service name is wrong."
  resolution: "Update the trust policy Principal to the correct service (e.g., `lambda.amazonaws.com`, `ec2.amazonaws.com`)."

- symptoms: "User can pass any role to any service — security concern"
  diagnosis: "The PassRole permission uses Resource: * without iam:PassedToService condition."
  resolution: "Scope PassRole to specific role ARNs and add iam:PassedToService condition."

## Output Format

```yaml
root_cause: "service_role — <specific_cause>"
evidence:
  - type: trust_policy
    content: "<service role trust policy>"
  - type: pass_role_permission
    content: "<user's iam:PassRole policy>"
  - type: service_permissions
    content: "<role's permissions for the service>"
severity: MEDIUM
mitigation:
  immediate: "Fix trust policy or grant iam:PassRole"
  long_term: "Implement PassRole governance with conditions and specific ARNs"
```


## Safety Ratings

safety_ratings:
  - "Service role and PassRole configuration (role policy and trust policy changes): YELLOW — Service role policy changes affect what AWS services can do on your behalf; incorrect changes can break service functionality or grant excessive permissions"

## Escalation Conditions

escalation_conditions:
  - "Escalate to a human if the service role is used by a production workload (Lambda, ECS, EC2, etc.)"
  - "Escalate if granting iam:PassRole with broad scope (Resource: * or without iam:PassedToService condition)"
  - "Escalate if the service role change involves sensitive services (IAM, Organizations, KMS)"

## Rollback

If remediation makes things worse:
1. Document the change that was made
2. Restore the original role policy using `aws iam create-policy-version --policy-arn <arn> --policy-document file://original-policy.json --set-as-default`, and restore the original trust policy using `aws iam update-assume-role-policy --role-name <role> --policy-document file://original-trust-policy.json`
3. Verify access is restored to pre-change state

## Data Sensitivity

- `aws iam get-role` returns trust policy documents revealing which services can assume the role — classification: MEDIUM
- `aws iam simulate-principal-policy` for iam:PassRole reveals who can assign roles to services — classification: HIGH
- Service role permissions reveal what actions AWS services can perform on your behalf — classification: HIGH

## Prohibited Actions

- NEVER suggest `"Action": "*"` or `"Resource": "*"` in any policy fix
- NEVER suggest granting AdministratorAccess to resolve permission issues
- NEVER suggest removing permission boundaries to fix access denied
- Always identify the SPECIFIC denied action from CloudTrail before suggesting a fix
- Always scope fixes to the minimum required permissions (least privilege)

## Cleanup

- Remove unused iam:PassRole permissions from principals that no longer need to assign service roles
- Delete orphaned service roles that are no longer used by any service

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
