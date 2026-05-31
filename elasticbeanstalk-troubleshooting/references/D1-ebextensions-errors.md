---
title: "D1 â€” .ebextensions Configuration Errors"
description: "Diagnose .ebextensions file parsing and execution failures"
status: active
severity: HIGH
triggers:
  - "ebextensions error"
  - "config file failed"
  - "container_commands failed"
  - "cfn-init error"
  - ".ebextensions"
owner: devops-agent
objective: "Fix .ebextensions configuration errors and ensure successful execution during deployment"
context: ".ebextensions files (.config YAML/JSON) customize the environment during deployment. They support packages, sources, files, users, groups, commands, container_commands, and services. Files execute in alphabetical order. Errors in any file fail the entire deployment."
---

## Phase 1 â€” Triage

MUST:
- List recent error events: `aws elasticbeanstalk describe-events --environment-name <env-name> --severity ERROR --max-records 20`
- Retrieve logs for cfn-init errors: `aws elasticbeanstalk request-environment-info --environment-name <env-name> --info-type bundle` then `aws elasticbeanstalk retrieve-environment-info --environment-name <env-name> --info-type bundle`
- Check CloudFormation stack events: `aws cloudformation describe-stack-events --stack-name awseb-<env-id>-stack --max-items 30`
- Verify the application version bundle structure: `aws elasticbeanstalk describe-application-versions --application-name <app-name> --version-labels <version-label>`

SHOULD:
- Check cfn-init.log and cfn-init-cmd.log in the log bundle for specific command failures
- Validate YAML/JSON syntax of .ebextensions files locally before deploying
- Check for platform-specific differences in .ebextensions support

MAY:
- Compare .ebextensions with a previously working version
- Check if commands require packages that haven't been installed yet (ordering issue)

## Phase 2 â€” Remediate

MUST:
- Fix YAML/JSON syntax errors in .ebextensions files
- Ensure commands are ordered correctly (alphabetical file naming)
- Fix failing commands or add `ignoreErrors: true` for non-critical commands

SHOULD:
- Use `test` directives in container_commands to conditionally run commands
- Use `leader_only: true` for one-time commands like database migrations
- Validate .ebextensions locally using a YAML linter before deploying

MAY:
- Migrate from .ebextensions to platform hooks (`.platform/hooks/`) on AL2+ platforms
- Use `.platform/confighooks/` for configuration-change-only hooks

## Common Issues

- symptoms: "YAML parsing error in .ebextensions config file"
  diagnosis: "Invalid YAML syntax â€” tabs instead of spaces, missing colons, or incorrect indentation."
  resolution: "Validate YAML syntax. Use spaces (not tabs) for indentation. Check for special characters."

- symptoms: "container_command failed with exit code 1"
  diagnosis: "The shell command returned a non-zero exit code."
  resolution: "Check cfn-init-cmd.log for the command output. Fix the command or add error handling."

- symptoms: "Package installation failed in .ebextensions"
  diagnosis: "Package not available in the repository or network connectivity issue."
  resolution: "Verify package name and availability. Check internet/NAT gateway connectivity for private subnets."

- symptoms: "File permission denied in .ebextensions commands"
  diagnosis: "Commands run as root by default but may reference files owned by other users."
  resolution: "Ensure file paths and permissions are correct. Use absolute paths in commands."

## Output Format

```yaml
root_cause: "ebextensions_error â€” <specific_cause>"
evidence:
  - type: events
    content: "<deployment error events>"
  - type: logs
    content: "<cfn-init.log and cfn-init-cmd.log entries>"
  - type: cloudformation
    content: "<stack events showing resource failure>"
severity: HIGH
mitigation:
  immediate: "Fix the .ebextensions error and redeploy"
  long_term: "Validate .ebextensions locally and consider migrating to platform hooks"
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
