---
title: "D2 â€” Environment Variable Issues"
description: "Diagnose environment variable configuration and access problems"
status: active
severity: MEDIUM
triggers:
  - "environment variable not set"
  - "env var missing"
  - "configuration variable"
  - "option settings"
  - "environment properties"
owner: devops-agent
objective: "Ensure environment variables are correctly configured and accessible to the application"
context: "Environment variables in Beanstalk are set via option settings in the aws:elasticbeanstalk:application:environment namespace. They are passed to the application process and available in .ebextensions. There is a combined limit on all option settings. Sensitive values should use Secrets Manager or SSM Parameter Store."
---

## Phase 1 â€” Triage

MUST:
- Check current environment variables: `aws elasticbeanstalk describe-configuration-settings --application-name <app-name> --environment-name <env-name> --query 'ConfigurationSettings[0].OptionSettings[?Namespace==`aws:elasticbeanstalk:application:environment`]'`
- Verify the variable is set in the environment configuration
- Check for option setting size limits (total option settings must be under 4096 bytes for the environment namespace)
- List recent events for configuration errors: `aws elasticbeanstalk describe-events --environment-name <env-name> --severity WARN --max-records 10`

SHOULD:
- Verify the application is reading the variable correctly (check process environment)
- Check if the variable was overridden by .ebextensions or platform hooks
- Verify the variable name does not conflict with reserved Beanstalk variables

MAY:
- Check if the variable is available in container_commands during deployment
- Review SSM Parameter Store or Secrets Manager if using dynamic references

## Phase 2 â€” Remediate

MUST:
- Set the environment variable via option settings: `aws elasticbeanstalk update-environment --environment-name <env-name> --option-settings Namespace=aws:elasticbeanstalk:application:environment,OptionName=<KEY>,Value=<VALUE>`
- Verify the variable is accessible after the environment update completes

SHOULD:
- Use Secrets Manager or SSM Parameter Store for sensitive values instead of plain-text environment variables
- Document all required environment variables for the application
- Use saved configurations to maintain consistent variable sets across environments

MAY:
- Use .ebextensions to set system-level environment variables in addition to Beanstalk properties
- Create a configuration template with all required variables

## Common Issues

- symptoms: "Environment variable is set but application cannot read it"
  diagnosis: "The variable is set in Beanstalk but not passed to the application process correctly."
  resolution: "Check platform-specific variable passing. Some platforms require Procfile or specific config."

- symptoms: "Environment update fails when setting variables"
  diagnosis: "Total option settings size exceeds the limit or variable name is invalid."
  resolution: "Reduce the number/size of environment variables. Use external configuration for large values."

- symptoms: "Variable value contains special characters that are mangled"
  diagnosis: "Special characters in values may be interpreted by the shell or platform."
  resolution: "Escape special characters or use base64 encoding for complex values."

## Output Format

```yaml
root_cause: "environment_variable_issue â€” <specific_cause>"
evidence:
  - type: configuration
    content: "<environment variable settings>"
  - type: application
    content: "<application variable access verification>"
severity: MEDIUM
mitigation:
  immediate: "Set or fix the environment variable configuration"
  long_term: "Use Secrets Manager for sensitive values and document required variables"
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
