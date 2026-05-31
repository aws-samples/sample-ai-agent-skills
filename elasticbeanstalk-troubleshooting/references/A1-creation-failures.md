---
title: "A1 â€” Environment Creation Failures"
description: "Diagnose why Elastic Beanstalk environment creation is failing"
status: active
severity: HIGH
triggers:
  - "environment creation failed"
  - "create environment error"
  - "environment launch failure"
  - "cannot create environment"
owner: devops-agent
objective: "Identify the root cause of environment creation failures and successfully launch the environment"
context: "Environment creation involves CloudFormation stack creation, EC2 instance launch, security group setup, ELB provisioning, and application deployment. Failures can occur at any stage and are best diagnosed through CloudFormation stack events and Beanstalk events."
---

## Phase 1 â€” Triage

MUST:
- Check environment status: `aws elasticbeanstalk describe-environments --environment-names <env-name>`
- List recent error events: `aws elasticbeanstalk describe-events --environment-name <env-name> --severity ERROR --max-records 20`
- Check CloudFormation stack events: `aws cloudformation describe-stack-events --stack-name awseb-<env-id>-stack --max-items 30`
- Verify the instance profile exists: `aws iam get-instance-profile --instance-profile-name aws-elasticbeanstalk-ec2-role`
- Verify the service role exists: `aws iam get-role --role-name aws-elasticbeanstalk-service-role`

SHOULD:
- Check platform version validity: `aws elasticbeanstalk describe-platform-version --platform-arn <platform-arn>`
- Verify VPC and subnet configuration: `aws ec2 describe-subnets --subnet-ids <subnet-ids>`
- Check security group existence: `aws ec2 describe-security-groups --group-ids <sg-ids>`
- Verify key pair exists: `aws ec2 describe-key-pairs --key-names <key-name>`

MAY:
- Check service quotas for EC2 instances: `aws service-quotas get-service-quota --service-code ec2 --quota-code L-1216C47A`
- Check account-level Beanstalk limits: `aws elasticbeanstalk describe-account-attributes`
- Review CloudTrail for creation events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=elasticbeanstalk.amazonaws.com --max-results 20`

## Phase 2 â€” Remediate

MUST:
- Fix the identified root cause (IAM roles, VPC config, platform version, quotas)
- Ensure both the instance profile and service role have required managed policies
- Retry environment creation after fixing the issue

SHOULD:
- Use `validate-configuration-settings` before creating to catch config errors early
- Ensure the application version bundle exists in the S3 source bucket
- Verify the solution stack name is valid and available in the region

MAY:
- Create a minimal environment first to isolate configuration issues
- Enable verbose CloudFormation logging for detailed failure analysis

## Common Issues

- symptoms: "Instance profile aws-elasticbeanstalk-ec2-role not found"
  diagnosis: "The default instance profile was not created. First-time Beanstalk setup requires role creation."
  resolution: "Create the instance profile with AWSElasticBeanstalkWebTier and AWSElasticBeanstalkWorkerTier policies."

- symptoms: "CloudFormation stack CREATE_FAILED â€” resource limit exceeded"
  diagnosis: "EC2 instance limit, EIP limit, or VPC resource limit reached."
  resolution: "Request a service quota increase for the specific resource type."

- symptoms: "Platform version is retired or not available"
  diagnosis: "The specified platform version has been retired by AWS."
  resolution: "Use `list-platform-versions` to find a current version in the same branch."

- symptoms: "Subnet not found or invalid VPC configuration"
  diagnosis: "The specified subnet or VPC does not exist or is in a different region."
  resolution: "Verify subnet IDs and VPC configuration match the environment's region."

## Output Format

```yaml
root_cause: "environment_creation_failure â€” <specific_cause>"
evidence:
  - type: environment_status
    content: "<describe-environments output>"
  - type: events
    content: "<error events from describe-events>"
  - type: cloudformation
    content: "<stack events showing failure>"
severity: HIGH
mitigation:
  immediate: "Fix the identified configuration issue and retry creation"
  long_term: "Validate configuration settings before environment creation"
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
