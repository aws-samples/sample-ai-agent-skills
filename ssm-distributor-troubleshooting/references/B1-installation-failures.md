---
title: "B1 — Installation Failures"
description: "Diagnose SSM Distributor package installation failures on managed instances"
status: active
severity: HIGH
triggers: ["installation failed", "package install error", "send-command failed", "install script error"]
owner: devops-agent
objective: "Resolve package installation failures on target instances"
context: "Installation involves downloading artifacts from S3, extracting them, and running platform-specific install scripts."
---

## Phase 1 — Triage

MUST:
- Check command status: `aws ssm list-command-invocations --command-id <command-id> --details --query 'CommandInvocations[*].{InstanceId:InstanceId,Status:Status,StatusDetails:StatusDetails}'`
- Get detailed output: `aws ssm get-command-invocation --command-id <command-id> --instance-id <instance-id>`
- Verify instance is managed: `aws ssm describe-instance-information --instance-information-filter-list key=InstanceIds,valueSet=<instance-id>`

SHOULD:
- Check instance platform matches package platform: `aws ssm describe-instance-information --instance-information-filter-list key=InstanceIds,valueSet=<instance-id> --query 'InstanceInformationList[0].PlatformType'`
- Verify SSM Agent version is current: minimum 2.3.274.0 for Distributor support

MAY:
- Check instance disk space and system logs via SSM Run Command
- Review CloudWatch Logs for SSM Agent logs if configured

## Phase 2 — Remediate

MUST:
- Fix install script errors identified in command output
- Ensure target instance has network access to S3 endpoints (VPC endpoint or NAT gateway)
- Verify IAM instance profile has s3:GetObject permission for the package bucket

SHOULD:
- Test installation on a single instance before deploying to fleet
- Increase command timeout for large packages: `--timeout-seconds 7200`

MAY:
- Enable verbose SSM Agent logging for detailed diagnostics
- Use S3 output for full command output: `--output-s3-bucket-name <bucket>`

## Common Issues

- symptoms: "Command status is Failed with 'Unable to download' error"
  diagnosis: "Instance cannot reach S3 to download package artifacts."
  resolution: "Verify VPC endpoint for S3 or NAT gateway. Check security groups and NACLs. Verify IAM permissions."

- symptoms: "Command status is Failed with exit code 1 from install script"
  diagnosis: "Install script encountered an error during execution."
  resolution: "Review StandardErrorContent for specific error. Common causes: missing dependencies, insufficient permissions, disk space."

## Safety Ratings

```
safety_ratings:
  - "list-command-invocations: GREEN — read-only command status"
  - "get-command-invocation: GREEN — read-only command output"
  - "describe-instance-information: GREEN — read-only instance metadata"
  - "send-command with Install action: YELLOW — installs software on instances"
```

## Escalation Conditions

- Installation fails across all instances with the same error after fixing known issues
- Installation succeeds but software does not function correctly post-install
- Installation causes instance instability or crashes
- Large-scale deployment failure affecting production fleet

## Data Sensitivity

- `aws ssm get-command-invocation`: MEDIUM — may contain application output and error details
- `aws ssm describe-instance-information`: LOW — returns instance metadata
- Install script output: MEDIUM — may contain system paths and configuration details
- S3 package artifacts: MEDIUM — contain installation binaries and scripts

## Prohibited Actions

- NEVER force-install packages on production instances without testing on staging first
- NEVER ignore install script errors and mark the issue as resolved
- NEVER install packages on instances with PingStatus ConnectionLost
- NEVER bypass IAM permission checks by using overly permissive policies

## Phase 3 — Rollback

1. Uninstall the failed package: `aws ssm send-command --document-name <package-name> --parameters action=Uninstall --targets Key=InstanceIds,Values=<instance-id>`
2. Verify uninstallation completed: `aws ssm get-command-invocation --command-id <uninstall-command-id> --instance-id <instance-id>`
3. If uninstall fails, manually clean up artifacts on the instance via Run Command
4. Restore any system configurations modified by the install script
5. Document the failure and root cause for the operations team

## Output Format

```yaml
root_cause: "installation_failure — <specific_cause>"
evidence:
  - type: command_output
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix install script or permissions"
  long_term: "Add pre-flight checks to install scripts"
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
