---
title: "F1 — Third-Party Package Issues"
description: "Diagnose SSM Distributor AWS-managed third-party package issues"
status: active
severity: MEDIUM
triggers: ["AmazonCloudWatchAgent", "AWSEC2Launch", "third-party package", "AWS managed package", "AWSPVDriver"]
owner: devops-agent
objective: "Resolve issues with AWS-managed third-party packages in Distributor"
context: "AWS-managed packages are owned by Amazon and cannot be modified. They include agents and drivers like AmazonCloudWatchAgent, AWSEC2Launch, AWSPVDriver."
---

## Phase 1 — Triage

MUST:
- List available AWS-managed packages: `aws ssm list-documents --document-filter-list key=Owner,value=Amazon key=DocumentType,value=Package --query 'DocumentIdentifiers[*].{Name:Name,PlatformTypes:PlatformTypes}'`
- Check package version: `aws ssm describe-document --name <package-name> --query '{LatestVersion:LatestVersion,DefaultVersion:DefaultVersion}'`
- Verify installation status on instance: `aws ssm list-command-invocations --instance-id <instance-id> --filters key=DocumentName,value=<package-name> --query 'CommandInvocations[*].{Status:Status,RequestedDateTime:RequestedDateTime}'`

SHOULD:
- Check available versions: `aws ssm list-document-versions --name <package-name> --query 'DocumentVersions[*].{Version:DocumentVersion,CreatedDate:CreatedDate}'`
- Verify the package is available in the target region

MAY:
- Check AWS documentation for known issues with specific package versions
- Review instance SSM Agent logs for package-specific errors

## Phase 2 — Remediate

MUST:
- Install the correct version for the target platform
- Verify post-installation configuration (e.g., CloudWatch Agent config file)
- Ensure the instance has network access to SSM and S3 endpoints

SHOULD:
- Pin to a specific version for production stability
- Test new versions on staging instances before production deployment

MAY:
- Set up associations for automatic version updates on a schedule
- Configure CloudWatch alarms for agent health monitoring

## Common Issues

- symptoms: "AmazonCloudWatchAgent installation succeeds but agent does not start"
  diagnosis: "CloudWatch Agent requires a configuration file after installation."
  resolution: "Create and apply a CloudWatch Agent configuration using amazon-cloudwatch-agent-ctl or SSM Parameter Store."

- symptoms: "Package version not available in the target region"
  diagnosis: "AWS-managed package versions may roll out to regions at different times."
  resolution: "Use the latest available version in the region. Check aws ssm list-document-versions for available versions."

## Safety Ratings

```
safety_ratings:
  - "list-documents with Owner=Amazon: GREEN — read-only package listing"
  - "describe-document: GREEN — read-only package metadata"
  - "send-command to install AWS package: YELLOW — installs AWS-managed software"
  - "Post-install configuration: YELLOW — configures installed software"
```

## Escalation Conditions

- AWS-managed package has a known bug affecting the target environment
- Package version required for compliance is not available in the region
- Installation succeeds but software malfunctions requiring AWS Support
- Fleet-wide upgrade of AWS-managed package causes widespread failures

## Data Sensitivity

- `aws ssm list-documents`: LOW — returns package metadata
- `aws ssm describe-document`: LOW — returns package version information
- `aws ssm list-command-invocations`: MEDIUM — reveals installation history
- Agent configuration files: MEDIUM — may contain monitoring targets and endpoints

## Prohibited Actions

- NEVER attempt to modify AWS-managed package content, scripts, or manifest
- NEVER downgrade AWS-managed packages without testing for compatibility
- NEVER install AWS-managed packages without verifying platform compatibility
- NEVER skip post-installation configuration for packages that require it

## Phase 3 — Rollback

1. If a new version causes issues, install the previous version: `aws ssm send-command --document-name <package-name> --document-version <previous-version> --parameters action=Install --targets Key=InstanceIds,Values=<instance-id>`
2. If the package must be removed, uninstall it: `aws ssm send-command --document-name <package-name> --parameters action=Uninstall --targets Key=InstanceIds,Values=<instance-id>`
3. Restore any pre-upgrade configuration files from backup
4. Verify the rollback restored the previous functionality
5. Report the issue to AWS Support if the package itself is defective

## Output Format

```yaml
root_cause: "third_party_package — <specific_cause>"
evidence:
  - type: package_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix version or configuration"
  long_term: "Pin versions and test before fleet deployment"
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
  - "NEVER suggest overly broad SSM document permissions"
  - "NEVER suggest disabling session logging"
  - "NEVER suggest sharing SSM documents publicly"
