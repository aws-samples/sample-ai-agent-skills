---
title: "D1 — Cross-Platform Packages"
description: "Diagnose SSM Distributor cross-platform package issues"
status: active
severity: MEDIUM
triggers: ["platform not supported", "wrong platform", "linux installer on windows", "manifest platform"]
owner: devops-agent
objective: "Resolve cross-platform package configuration and deployment issues"
context: "Distributor packages support Windows, Linux, and macOS. Each platform requires its own installer definition in manifest.json."
---

## Phase 1 — Triage

MUST:
- Check package platform support: `aws ssm describe-document --name <package-name> --query 'Document.PlatformTypes'`
- Verify target instance platform: `aws ssm describe-instance-information --instance-information-filter-list key=InstanceIds,valueSet=<instance-id> --query 'InstanceInformationList[0].{PlatformType:PlatformType,PlatformName:PlatformName,PlatformVersion:PlatformVersion}'`
- Compare instance platform with package supported platforms

SHOULD:
- Review manifest.json for platform-specific entries
- Check if install/uninstall scripts exist for the target platform

MAY:
- List all instances by platform to identify scope: `aws ssm describe-instance-information --query 'InstanceInformationList[*].{InstanceId:InstanceId,PlatformType:PlatformType}' --output table`

## Phase 2 — Remediate

MUST:
- Add missing platform installer to the package manifest
- Ensure install/uninstall scripts are compatible with the target OS
- Create a new package version with the platform-specific content

SHOULD:
- Test on each target platform before fleet deployment
- Use platform-appropriate package formats (MSI for Windows, RPM/DEB for Linux)

MAY:
- Create separate packages per platform if cross-platform complexity is too high
- Use conditional logic in scripts to handle OS-specific differences

## Common Issues

- symptoms: "Installation fails with 'platform not supported' on Linux instances"
  diagnosis: "Package manifest does not include a linux platform entry."
  resolution: "Add linux platform entry to manifest.json with appropriate install/uninstall scripts and artifacts."

- symptoms: "Install script fails on Amazon Linux 2023 but works on Amazon Linux 2"
  diagnosis: "Script uses OS-specific commands not available on the target distribution."
  resolution: "Update install script to detect OS version and use appropriate commands (e.g., dnf vs yum)."

## Safety Ratings

```
safety_ratings:
  - "describe-document: GREEN — read-only package metadata"
  - "describe-instance-information: GREEN — read-only instance metadata"
  - "update-document with new platform: YELLOW — creates new package version"
  - "send-command for testing: YELLOW — installs software on test instances"
```

## Escalation Conditions

- Package must support a platform not currently in the manifest and requires new installer development
- Cross-platform script differences cause inconsistent behavior across OS types
- Platform-specific dependencies are unavailable in the target environment
- Large mixed-OS fleet requires coordinated cross-platform deployment

## Data Sensitivity

- `aws ssm describe-document`: LOW — returns package metadata
- `aws ssm describe-instance-information`: LOW — returns instance platform details
- manifest.json content: LOW — contains package structure metadata
- Install scripts: MEDIUM — may contain system configuration logic

## Prohibited Actions

- NEVER deploy a package to a platform not listed in its PlatformTypes
- NEVER assume Linux scripts work on macOS or vice versa without testing
- NEVER use Windows-specific commands (PowerShell) in Linux install scripts
- NEVER skip platform-specific testing before fleet deployment

## Phase 3 — Rollback

1. If a cross-platform package was deployed to the wrong platform, uninstall it: `aws ssm send-command --document-name <package-name> --parameters action=Uninstall --targets Key=InstanceIds,Values=<instance-id>`
2. Revert to the previous package version if the new cross-platform version has issues
3. If platform-specific scripts caused system changes, restore from backup
4. Verify all target platforms are functioning correctly after rollback
5. Document platform compatibility matrix for the package

## Output Format

```yaml
root_cause: "cross_platform — <specific_cause>"
evidence:
  - type: platform_mismatch
    content: "<platform details>"
severity: MEDIUM
mitigation:
  immediate: "Add platform support or use correct package"
  long_term: "Maintain platform compatibility matrix"
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
