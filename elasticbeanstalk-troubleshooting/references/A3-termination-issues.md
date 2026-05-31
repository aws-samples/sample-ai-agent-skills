---
title: "A3 â€” Environment Termination Issues"
description: "Diagnose why Elastic Beanstalk environment termination is failing or incomplete"
status: active
severity: MEDIUM
triggers:
  - "cannot terminate environment"
  - "termination stuck"
  - "environment won't delete"
  - "resources left after termination"
owner: devops-agent
objective: "Successfully terminate the environment and clean up all associated resources"
context: "Environment termination deletes the CloudFormation stack and all managed resources. Failures can occur due to resource dependencies, termination protection, coupled RDS instances, or CloudFormation stack deletion failures."
---

## Phase 1 â€” Triage

MUST:
- Check environment status: `aws elasticbeanstalk describe-environments --environment-names <env-name>`
- Check if termination protection is enabled: `aws elasticbeanstalk describe-environments --environment-names <env-name> --query 'Environments[0].{Status:Status,Protection:EnvironmentLinks}'`
- Check CloudFormation stack status: `aws cloudformation describe-stacks --stack-name awseb-<env-id>-stack`
- List stack resources for deletion failures: `aws cloudformation list-stack-resources --stack-name awseb-<env-id>-stack`
- Check for coupled RDS instance: `aws elasticbeanstalk describe-configuration-settings --application-name <app-name> --environment-name <env-name> --query 'ConfigurationSettings[0].OptionSettings[?Namespace==`aws:rds:dbinstance`]'`

SHOULD:
- Check CloudFormation stack events for deletion errors: `aws cloudformation describe-stack-events --stack-name awseb-<env-id>-stack --max-items 20`
- Verify no other environments depend on shared resources
- Check for S3 bucket deletion issues (non-empty buckets)

MAY:
- Check for ENIs or security groups with external dependencies
- Review CloudTrail for termination events

## Phase 2 â€” Remediate

MUST:
- Disable termination protection if enabled before retrying
- Handle coupled RDS instance (snapshot or delete) before environment termination
- If CloudFormation stack is stuck in DELETE_FAILED, identify and manually remove blocking resources

SHOULD:
- Create an RDS snapshot before terminating environments with coupled databases
- Clean up orphaned resources (security groups, EIPs) after termination
- Verify all resources are deleted after termination completes

MAY:
- Use `aws cloudformation delete-stack --retain-resources` to skip problematic resources
- Manually delete the CloudFormation stack if Beanstalk termination is stuck

## Common Issues

- symptoms: "Environment stuck in Terminating state"
  diagnosis: "CloudFormation stack deletion is failing on a specific resource."
  resolution: "Check stack events for the failing resource. Manually remove dependencies and retry."

- symptoms: "Cannot terminate â€” RDS instance deletion blocked"
  diagnosis: "Coupled RDS instance has deletion protection or a final snapshot requirement."
  resolution: "Create a final snapshot, disable deletion protection, then retry termination."

- symptoms: "Security group cannot be deleted â€” in use by another resource"
  diagnosis: "Another resource (e.g., Lambda, RDS) references the Beanstalk security group."
  resolution: "Remove the external dependency first, then retry environment termination."

## Output Format

```yaml
root_cause: "termination_failure â€” <specific_cause>"
evidence:
  - type: environment_status
    content: "<describe-environments output>"
  - type: cloudformation
    content: "<stack deletion events>"
  - type: resources
    content: "<blocking resources>"
severity: MEDIUM
mitigation:
  immediate: "Remove blocking dependencies and retry termination"
  long_term: "Decouple RDS instances and avoid external security group references"
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
