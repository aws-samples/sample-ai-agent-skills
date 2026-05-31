---
title: "A4 â€” Platform Version Issues"
description: "Diagnose platform version upgrade, retirement, and compatibility problems"
status: active
severity: MEDIUM
triggers:
  - "platform version retired"
  - "platform upgrade failed"
  - "unsupported platform"
  - "managed update failed"
  - "platform branch"
owner: devops-agent
objective: "Resolve platform version issues and ensure the environment runs on a supported, current platform"
context: "Elastic Beanstalk platforms have branches (e.g., Python 3.9 on AL2) and versions within branches. AWS retires old versions and branches. Managed platform updates move between versions within a branch but cannot change branches. Branch migration requires blue/green deployment."
---

## Phase 1 â€” Triage

MUST:
- Check current platform version: `aws elasticbeanstalk describe-environments --environment-names <env-name> --query 'Environments[0].PlatformArn'`
- Describe the platform version: `aws elasticbeanstalk describe-platform-version --platform-arn <platform-arn>`
- List available platform versions for the branch: `aws elasticbeanstalk list-platform-versions --filters "Type=PlatformBranchName,Operator==,Values=<branch-name>"`
- Check managed update configuration: `aws elasticbeanstalk describe-configuration-settings --application-name <app-name> --environment-name <env-name> --query 'ConfigurationSettings[0].OptionSettings[?Namespace==`aws:elasticbeanstalk:managedactions:platformupdate`]'`

SHOULD:
- List all platform branches: `aws elasticbeanstalk list-platform-branches`
- Check if the current platform is marked as retired
- Review recent events for managed update failures: `aws elasticbeanstalk describe-events --environment-name <env-name> --severity WARN --max-records 20`

MAY:
- Compare configuration options between current and target platform versions
- Check for deprecated configuration options in the target version

## Phase 2 â€” Remediate

MUST:
- If platform version is retired, upgrade to a supported version within the same branch
- If branch migration is needed, plan a blue/green deployment to the new branch
- Test the application on the target platform version before production migration

SHOULD:
- Enable managed platform updates for automatic version maintenance
- Use immutable update type for managed updates to enable rollback
- Document platform-specific configuration differences

MAY:
- Create a staging environment on the target platform for testing
- Review AWS release notes for breaking changes in the target version

## Common Issues

- symptoms: "Platform version is retired â€” environment cannot be updated"
  diagnosis: "The current platform version has been retired by AWS."
  resolution: "Update to the latest version in the same branch using `update-environment --platform-arn`."

- symptoms: "Managed platform update failed with health check errors"
  diagnosis: "The new platform version introduced changes incompatible with the application."
  resolution: "Test on a staging environment. Fix compatibility issues before retrying the update."

- symptoms: "Cannot change from Amazon Linux 2 to Amazon Linux 2023 via update"
  diagnosis: "Branch changes cannot be done in-place. This requires blue/green deployment."
  resolution: "Create a new environment on AL2023, test, then swap CNAMEs."

## Output Format

```yaml
root_cause: "platform_version_issue â€” <specific_cause>"
evidence:
  - type: platform
    content: "<current platform ARN and status>"
  - type: available_versions
    content: "<available versions in the branch>"
  - type: events
    content: "<managed update events>"
severity: MEDIUM
mitigation:
  immediate: "Upgrade to a supported platform version"
  long_term: "Enable managed updates and test platform changes in staging"
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
