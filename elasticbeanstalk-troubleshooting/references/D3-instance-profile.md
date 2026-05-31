---
title: "D3 â€” Instance Profile and IAM Issues"
description: "Diagnose instance profile and IAM role configuration problems"
status: active
severity: HIGH
triggers:
  - "instance profile"
  - "IAM role"
  - "access denied"
  - "permission denied"
  - "service role"
owner: devops-agent
objective: "Configure correct IAM roles and instance profiles for Beanstalk operations"
context: "Elastic Beanstalk requires two IAM roles: a service role (for Beanstalk service operations) and an instance profile (for EC2 instances). Missing or misconfigured roles cause environment creation failures, deployment errors, and application permission issues."
---

## Phase 1 â€” Triage

MUST:
- Check instance profile configuration: `aws elasticbeanstalk describe-configuration-settings --application-name <app-name> --environment-name <env-name> --query 'ConfigurationSettings[0].OptionSettings[?Namespace==`aws:autoscaling:launchconfiguration` && OptionName==`IamInstanceProfile`]'`
- Verify the instance profile exists: `aws iam get-instance-profile --instance-profile-name <profile-name>`
- Check the role attached to the instance profile: `aws iam list-instance-profiles-for-role --role-name <role-name>`
- Check the service role: `aws iam get-role --role-name aws-elasticbeanstalk-service-role`
- List role policies: `aws iam list-attached-role-policies --role-name <role-name>`

SHOULD:
- Verify managed policies are attached: AWSElasticBeanstalkWebTier, AWSElasticBeanstalkWorkerTier, AWSElasticBeanstalkMulticontainerDocker
- Check for custom inline policies: `aws iam list-role-policies --role-name <role-name>`
- Verify the service role has AWSElasticBeanstalkEnhancedHealth and AWSElasticBeanstalkManagedUpdatesCustomerRolePolicy

MAY:
- Check CloudTrail for AccessDenied events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=AssumeRole --max-results 20`
- Verify the trust policy allows ec2.amazonaws.com to assume the role

## Phase 2 â€” Remediate

MUST:
- Create the instance profile if missing with the required managed policies
- Ensure the trust policy allows `ec2.amazonaws.com` for instance profiles
- Ensure the service role trust policy allows `elasticbeanstalk.amazonaws.com`

SHOULD:
- Use AWS managed policies rather than custom policies for standard operations
- Add application-specific permissions (S3, SQS, DynamoDB) to the instance role
- Follow least-privilege principle for custom permissions

MAY:
- Create separate instance profiles for different environments (dev, staging, prod)
- Use IAM Access Analyzer to identify required permissions

## Common Issues

- symptoms: "Instance profile not found during environment creation"
  diagnosis: "The default instance profile was never created (first-time Beanstalk setup)."
  resolution: "Create the instance profile and role with AWSElasticBeanstalkWebTier policy attached."

- symptoms: "Application cannot access S3/SQS/DynamoDB from Beanstalk instances"
  diagnosis: "The instance profile role lacks permissions for the target AWS service."
  resolution: "Add the required service permissions to the instance profile role."

- symptoms: "Enhanced health not working â€” permission denied"
  diagnosis: "The service role lacks the AWSElasticBeanstalkEnhancedHealth managed policy."
  resolution: "Attach the AWSElasticBeanstalkEnhancedHealth policy to the service role."

## Output Format

```yaml
root_cause: "iam_configuration_issue â€” <specific_cause>"
evidence:
  - type: instance_profile
    content: "<instance profile and role details>"
  - type: policies
    content: "<attached policies>"
  - type: service_role
    content: "<service role configuration>"
severity: HIGH
mitigation:
  immediate: "Fix IAM role configuration and permissions"
  long_term: "Use managed policies and follow least-privilege principle"
```

---

## Safety Ratings

| Aspect | Rating |
|--------|--------|
| Service Impact | HIGH — Elastic Beanstalk manages full application stacks including compute, networking, and load balancing |
| Data Risk | MEDIUM — Environment changes can affect application availability and data connections |
| Recovery Complexity | MEDIUM — Environments can be rebuilt but require correct configuration and deployment artifacts |
| Blast Radius | HIGH — Environment-level changes affect all instances and the application endpoint |

## Escalation Conditions

- Environment health degraded to Red with customer-facing impact
- Deployment failure with no successful rollback available
- Auto-scaling events failing causing capacity shortage under load
- Platform update causing application incompatibility in production
- CloudFormation stack in UPDATE_ROLLBACK_FAILED state requiring manual intervention
- Load balancer health checks failing across all instances

## Data Sensitivity

- Environment variables may contain database credentials, API keys, and secrets
- Instance profile roles may grant broad AWS service access
- Application source bundles in S3 contain proprietary application code
- Log files may contain PII, authentication tokens, or business data
- .ebextensions may contain infrastructure secrets and custom configurations

## PROHIBITED Actions

- NEVER suggest terminating a production environment without confirming blue-green swap is ready
- NEVER modify environment variables containing secrets without secure handling procedures
- NEVER force a platform update on production without testing on a staging environment first
- NEVER delete application versions that are currently deployed to any environment
- NEVER change the load balancer type on a running environment (requires environment rebuild)
- NEVER disable enhanced health reporting on production environments

## Phase 3 — Rollback

- If deployment failed: use ws elasticbeanstalk abort-environment-update then redeploy last known good version
- If environment configuration change caused issues: apply previous saved configuration via update-environment
- If platform update broke the application: rebuild environment on previous platform version
- If auto-scaling changes caused instability: revert min/max/desired counts to previous values
- If .ebextensions changes caused failures: redeploy previous application version with working extensions
- Document all rollback actions with environment ID, version label, and configuration template name
## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN â€” read-only"
  - "Phase 2 configuration changes: YELLOW â€” state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED â€” destructive or irreversible"

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
