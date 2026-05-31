---
title: "D1 — Package Publishing Failures"
description: "Diagnose CodeArtifact package publishing failures including permissions, format, and size issues"
status: active
severity: HIGH
triggers:
  - "publish failed"
  - "npm publish codeartifact"
  - "twine upload failed"
  - "mvn deploy failed"
  - "package publish error"
owner: devops-agent
objective: "Resolve package publishing failures to CodeArtifact"
context: "Publishing packages to CodeArtifact requires proper auth, IAM permissions (codeartifact:PublishPackageVersion), correct endpoint configuration, and valid package format. Package versions are immutable once published."
---

## Phase 1 — Triage

MUST:
- Verify auth token is current and has publish permissions
- Check IAM permissions include `codeartifact:PublishPackageVersion`
- Verify repository endpoint is correct for the package format
- Check if version already exists (immutable): `aws codeartifact list-package-versions --domain <domain> --repository <repo> --package <pkg> --format <format>`

SHOULD:
- Verify package format matches repository configuration
- Check asset size (max 5 GB per asset)
- Check CloudTrail for publish errors

MAY:
- Verify package metadata is valid for the format
- Check for repository permissions policy restrictions

## Phase 2 — Remediate

MUST:
- If version exists, publish with new version number or delete first: `aws codeartifact delete-package-versions --domain <domain> --repository <repo> --package <pkg> --format <format> --versions <version>`
- If auth error, refresh token with publish permissions
- If permission denied, add `codeartifact:PublishPackageVersion` to IAM policy

SHOULD:
- Use `aws codeartifact login` with the appropriate tool flag
- Validate package locally before publishing

## Output Format

```yaml
root_cause: "publishing_failure — <specific_cause>"
evidence:
  - type: publish_error
    content: "<error message>"
  - type: version_check
    content: "<existing versions>"
severity: HIGH
mitigation:
  immediate: "Fix auth, permissions, or version conflict"
  long_term: "Automate publishing in CI/CD with version validation"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Publish with new version number | GREEN |
| Delete existing version then republish | RED |
| Refresh token with publish permissions | GREEN |
| Add PublishPackageVersion to IAM policy | YELLOW |
| Validate package locally before publishing | GREEN |

## Escalation Conditions

- Fix requires deleting existing package versions in production repositories
- Fix involves modifying IAM permissions for package publishing

## Data Sensitivity

- **HIGH**: package publishing affects all consumers of the repository
- Deleting and republishing versions can introduce supply chain risks

## Prohibited Actions

- NEVER suggest deleting upstream connections without understanding dependency impact
- NEVER suggest publishing over existing package versions

## Phase 3 — Rollback

1. If a package version was deleted, it cannot be restored — publish the correct version
2. If IAM permissions were modified, revert to previous policy
3. If auth token configuration was changed, restore previous settings
4. Verify rollback by confirming package versions match expected state

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

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
