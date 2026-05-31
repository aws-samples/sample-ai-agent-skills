---
title: "F1 — Access Control Issues"
description: "Diagnose Amplify access control and permission problems"
status: active
severity: MEDIUM
triggers:
  - "access denied"
  - "permission error"
  - "IAM role issue"
  - "service role"
  - "access control"
owner: devops-agent
objective: "Identify and resolve access control issues"
context: "Access control issues include: IAM service role missing permissions, Amplify console access restrictions, repository access denied, or backend resource permissions insufficient."
---

## Phase 1 — Triage

MUST:
- Check Amplify service role: `aws amplify get-app --app-id <app-id> --query 'app.iamServiceRoleArn'`
- Verify service role permissions: `aws iam get-role --role-name <role-name>`
- Check role policies: `aws iam list-attached-role-policies --role-name <role-name>`
- Review error messages for specific permission denials

SHOULD:
- Check if service role has AdministratorAccess-Amplify managed policy
- Verify repository access token is valid
- Check for SCP restrictions in AWS Organizations

MAY:
- Review CloudTrail for access denied events
- Check resource-based policies on backend resources

## Phase 2 — Remediate

MUST:
- Attach required policies to service role
- Update repository access token if expired
- Fix specific permission denials

SHOULD:
- Use Amplify managed policy: `aws iam attach-role-policy --role-name <role> --policy-arn arn:aws:iam::aws:policy/AdministratorAccess-Amplify`
- Create service role if missing

MAY:
- Implement least-privilege custom policy
- Set up role monitoring for permission changes

## Safety Ratings

```
safety_ratings:
  - "get-app (iamServiceRoleArn): GREEN — read-only role check"
  - "list-attached-role-policies: GREEN — read-only policy check"
  - "attach-role-policy: YELLOW — grants additional permissions"
  - "create service role: YELLOW — creates new IAM role"
```

## Escalation Conditions

- Service role missing permissions blocking production deployments
- Repository access token expired, preventing all builds
- SCP restrictions blocking Amplify operations
- Service role needs permissions that require security team approval

## Data Sensitivity

- Service role ARN and policies: MEDIUM — reveals IAM permissions
- Repository access token: HIGH — provides source code access
- Permission error details: MEDIUM — reveals denied actions

## Prohibited Actions

- NEVER suggest deleting a production branch
- NEVER suggest disabling HTTPS
- NEVER attach AdministratorAccess to the Amplify service role
- NEVER store repository access tokens in environment variables

## Phase 3 — Rollback

1. If IAM policies were changed incorrectly, detach them: `aws iam detach-role-policy --role-name <role> --policy-arn <policy-arn>`
2. If a service role was created incorrectly, delete it and restore the previous role
3. If repository access was changed, restore previous access token
4. Verify builds and deployments work correctly after rollback
5. Document IAM changes for security audit

## Output Format

```yaml
root_cause: "access_control_issue — <specific_cause>"
evidence:
  - type: service_role
    content: "<role ARN and policies>"
  - type: permission_error
    content: "<specific denied action>"
severity: MEDIUM
mitigation:
  immediate: "Fix IAM permissions"
  long_term: "Implement proper service role with required permissions"
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
