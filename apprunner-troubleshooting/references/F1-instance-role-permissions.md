---
title: "F1 — App Runner Instance Role Permission Issues"
description: "Diagnose IAM instance role configuration and permission errors"
status: active
severity: HIGH
triggers:
  - "instance role"
  - "access denied"
  - "permission denied"
  - "IAM role"
  - "cannot access S3"
  - "cannot access DynamoDB"
  - "credentials"
owner: devops-agent
objective: "Identify and fix instance role permission issues for App Runner services"
context: "The instance role provides AWS credentials to the running application via the AWS SDK. The role must have a trust policy allowing apprunner.amazonaws.com to assume it. Without an instance role, the application cannot make AWS API calls. Common issues include missing trust policy, insufficient permissions, and confusion between the access role (for ECR) and the instance role (for runtime)."
---

## Phase 1 — Triage

MUST:
- Check instance role configuration: `aws apprunner describe-service --service-arn <service-arn> --query 'Service.InstanceConfiguration.InstanceRoleArn'`
- If instance role is set, check the trust policy: `aws iam get-role --role-name <role-name> --query 'Role.AssumeRolePolicyDocument'`
- Check attached policies: `aws iam list-attached-role-policies --role-name <role-name>`
- Check inline policies: `aws iam list-role-policies --role-name <role-name>`
- Review application logs for access denied errors: `aws logs get-log-events --log-group-name /aws/apprunner/<service-name>/<service-id>/application --log-stream-name <stream>`

SHOULD:
- Verify the trust policy includes `{"Service": "apprunner.amazonaws.com"}` as a trusted principal
- Check if the role has the specific permissions needed by the application (S3, DynamoDB, SQS, etc.)
- Distinguish between the access role (for ECR image pull) and the instance role (for runtime AWS access)

MAY:
- Use IAM Access Analyzer to identify missing permissions
- Check CloudTrail for AccessDenied events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=AssumeRole --max-results 10`
- Verify no SCP or permission boundary is blocking the required actions

## Phase 2 — Remediate

MUST:
- Ensure the instance role trust policy allows apprunner.amazonaws.com:
  ```json
  {
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "apprunner.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }
  ```
- Add the required IAM permissions for the AWS services the application needs
- If no instance role is configured, create one and update the service

SHOULD:
- Follow least-privilege principle — only grant permissions the application actually needs
- Use managed policies where appropriate (e.g., AmazonS3ReadOnlyAccess)
- Add a condition key to restrict the role to specific App Runner services if needed

MAY:
- Implement IAM Access Analyzer recommendations
- Set up CloudTrail monitoring for permission denied events
- Use resource-based policies on target services as an alternative where supported

## Common Issues

- symptoms: "Application gets 'Unable to locate credentials' error"
  diagnosis: "No instance role is configured on the App Runner service."
  resolution: "Create an IAM role with apprunner.amazonaws.com trust policy and attach it to the service."

- symptoms: "Application gets 'AccessDenied' for specific AWS API calls"
  diagnosis: "Instance role exists but lacks the required permissions."
  resolution: "Add the missing IAM permissions to the instance role."

- symptoms: "Instance role cannot be attached — 'InvalidParameterException'"
  diagnosis: "Role trust policy does not include apprunner.amazonaws.com."
  resolution: "Update the role trust policy to allow apprunner.amazonaws.com to assume the role."

## Output Format

```yaml
root_cause: "instance_role — <specific_cause>"
evidence:
  - type: instance_role
    content: "<role ARN and trust policy>"
  - type: permissions
    content: "<attached and inline policies>"
  - type: application_logs
    content: "<access denied error details>"
severity: HIGH
mitigation:
  immediate: "Fix trust policy or add required permissions"
  long_term: "Implement least-privilege permissions and monitor with CloudTrail"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Ensure instance role trust policy correct | YELLOW | IAM trust policy change |
| Add required IAM permissions | YELLOW | IAM policy change |
| Create instance role if none exists | YELLOW | New IAM role creation |
| Follow least-privilege principle | GREEN | Security best practice |
| Use managed policies | GREEN | Policy selection |

## Escalation Conditions

- Fix requires redeployment of production service
- NEVER suggest scaling to 0 (not supported)

## Data Sensitivity

- **Instance role**: IAM role ARN and trust policy — INTERNAL
- **IAM permissions**: Attached and inline policies — CONFIDENTIAL
- **Application logs**: Access denied error details — INTERNAL

## Prohibited Actions

- NEVER suggest hardcoding credentials in environment variables
- NEVER suggest scaling to 0 instances (not supported by App Runner)
- NEVER grant AdministratorAccess to an instance role

## Phase 3 — Rollback

1. Revert IAM policy changes to previous version
2. Revert trust policy changes if modified
3. Detach newly attached policies if they caused issues
4. Verify application can still access required AWS services
5. Monitor CloudTrail for access denied events after rollback

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
