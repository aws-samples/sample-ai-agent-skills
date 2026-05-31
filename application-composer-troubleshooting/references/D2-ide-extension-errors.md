---
title: "D2 — IDE Extension Errors"
description: "Diagnose IDE extension errors for Application Composer"
status: active
severity: MEDIUM
triggers: ["IDE extension error", "VS Code extension not working", "AWS Toolkit error", "extension crash"]
owner: devops-agent
objective: "Resolve IDE extension issues for Application Composer integration"
context: "The VS Code AWS Toolkit extension provides Application Composer integration. Extension errors often stem from credential or configuration issues."
---

## Phase 1 — Triage

MUST:
- Verify AWS credentials are configured: `aws sts get-caller-identity`
- Check AWS Toolkit extension version in VS Code Extensions panel
- Review VS Code Output panel for AWS Toolkit errors

SHOULD:
- Verify SSO session is active if using SSO: `aws sso login --profile <profile>`
- Check VS Code version compatibility with the extension

MAY:
- Review extension logs: `~/.vscode/extensions/amazonwebservices.aws-toolkit-vscode-*/logs/`
- Check for conflicting extensions

## Phase 2 — Remediate

MUST:
- Refresh expired AWS credentials or SSO session
- Update the AWS Toolkit extension to the latest version
- Restart VS Code after credential changes

SHOULD:
- Configure the correct AWS profile in the extension settings
- Verify the AWS region is set correctly in the extension

MAY:
- Reinstall the extension if persistent errors occur
- Clear extension cache and workspace settings

## Safety Ratings

```
safety_ratings:
  - "aws sts get-caller-identity: GREEN — read-only identity check"
  - "aws sso login: GREEN — refreshes SSO session token"
  - "extension reinstall: YELLOW — removes extension settings"
  - "VS Code restart: GREEN — no data loss"
```

## Escalation Conditions

- Extension crashes repeatedly after reinstallation
- Credentials are valid but extension cannot authenticate
- Extension conflicts with other installed VS Code extensions
- Application Composer canvas fails to render in the IDE

## Data Sensitivity

- `aws sts get-caller-identity`: LOW — reveals account ID and IAM principal
- Extension logs: MEDIUM — may contain API call details and error context
- AWS profile configuration: MEDIUM — reveals account and region settings

## Prohibited Actions

- NEVER store AWS credentials in plain text in VS Code settings
- NEVER share extension logs without reviewing for sensitive information
- NEVER use root account credentials with the IDE extension
- NEVER disable VS Code security features to fix extension issues

## Phase 3 — Rollback

1. If extension settings were modified, restore from VS Code settings sync or backup
2. If extension was reinstalled, reconfigure AWS profile and region settings
3. If SSO session was refreshed, verify the correct account and role are selected
4. Restore workspace settings from version control: `git checkout -- .vscode/settings.json`
5. Verify the extension is functional by opening an Application Composer template

## Common Issues

- symptoms: "Application Composer panel shows 'Unable to load' in VS Code"
  diagnosis: "AWS credentials are expired or the AWS Toolkit extension cannot authenticate."
  resolution: "Refresh credentials: aws sso login or update access keys. Restart VS Code."

## Output Format

```yaml
root_cause: "ide_extension — <specific_cause>"
evidence:
  - type: extension_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix credentials or extension configuration"
  long_term: "Use SSO with automatic token refresh"
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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
