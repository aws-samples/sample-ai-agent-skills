---
title: "A2 — Package Version Errors"
description: "Diagnose SSM Distributor package version management issues"
status: active
severity: MEDIUM
triggers: ["wrong package version", "version not found", "update-document failed", "default version"]
owner: devops-agent
objective: "Resolve package version management issues"
context: "Package versions are immutable. Default version determines which version is installed when no specific version is requested."
---

## Phase 1 — Triage

MUST:
- List package versions: `aws ssm list-document-versions --name <package-name> --query 'DocumentVersions[*].{Version:DocumentVersion,IsDefault:IsDefaultVersion,Status:Status}'`
- Check current default version: `aws ssm describe-document --name <package-name> --query '{DefaultVersion:DefaultVersion,LatestVersion:LatestVersion}'`
- Verify the target version exists and is not in a failed state

SHOULD:
- Compare version content between expected and actual: `aws ssm get-document --name <package-name> --document-version <version>`
- Check if associations reference a specific version or the default

MAY:
- Review CloudTrail for UpdateDocument and UpdateDocumentDefaultVersion events

## Phase 2 — Remediate

MUST:
- Create a new version if the current version has issues: `aws ssm update-document --name <package-name> --content file://manifest.json --document-version '$LATEST' --attachments Key=SourceUrl,Values=s3://<bucket>/packages/`
- Update default version if needed: `aws ssm update-document-default-version --name <package-name> --document-version <version>`

SHOULD:
- Verify associations pick up the new default version on next execution
- Test the new version on a single instance before broad deployment

MAY:
- Implement version naming conventions for tracking

## Common Issues

- symptoms: "Association installs old version despite new version being created"
  diagnosis: "Association targets a specific version number, not the default."
  resolution: "Update the association to use the new version or set the new version as default and ensure association uses default."

## Safety Ratings

```
safety_ratings:
  - "list-document-versions: GREEN — read-only version enumeration"
  - "describe-document: GREEN — read-only document metadata"
  - "update-document: YELLOW — creates new immutable version"
  - "update-document-default-version: YELLOW — changes which version is used by default"
```

## Escalation Conditions

- Version creation fails with internal errors after validating content
- Default version change does not propagate to running associations
- Multiple versions exist with unclear lineage causing deployment confusion
- Version rollback needed across hundreds of instances

## Data Sensitivity

- `aws ssm list-document-versions`: LOW — returns version metadata
- `aws ssm get-document`: MEDIUM — returns full package content including scripts
- `aws ssm update-document`: MEDIUM — modifies package content
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history

## Prohibited Actions

- NEVER delete package versions that are referenced by active associations
- NEVER change the default version in production without testing
- NEVER assume the latest version is the default version — they can differ
- NEVER modify package content by deleting and recreating with the same version number (versions are immutable)

## Phase 3 — Rollback

1. Revert default version: `aws ssm update-document-default-version --name <package-name> --document-version <previous-version>`
2. If a bad version was deployed, send a command to reinstall the previous version: `aws ssm send-command --document-name <package-name> --document-version <good-version> --targets Key=tag:Environment,Values=<env> --parameters action=Install`
3. Verify rollback by checking command execution status on target instances
4. Update associations to reference the correct version
5. Document version history and rollback reason

## Output Format

```yaml
root_cause: "package_version — <specific_cause>"
evidence:
  - type: version_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix version or revert default"
  long_term: "Implement version management process"
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
