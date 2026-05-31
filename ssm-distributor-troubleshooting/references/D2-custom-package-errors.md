---
title: "D2 — Custom Package Errors"
description: "Diagnose SSM Distributor custom package creation and deployment errors"
status: active
severity: HIGH
triggers: ["custom package error", "manifest.json invalid", "package attachment", "custom installer"]
owner: devops-agent
objective: "Resolve custom package creation, manifest, and deployment issues"
context: "Custom packages require a valid manifest.json, S3-hosted artifacts, and platform-specific install/uninstall scripts."
---

## Phase 1 — Triage

MUST:
- Verify package document status: `aws ssm describe-document --name <package-name> --query '{Status:Status,DocumentVersion:DocumentVersion}'`
- Check manifest.json structure for required fields (schemaVersion, version, packages, files)
- Verify S3 artifacts exist: `aws s3 ls s3://<bucket>/packages/<package-name>/`

SHOULD:
- Validate install/uninstall script syntax for each platform
- Check S3 bucket region matches target instance region
- Verify file checksums in manifest match actual S3 artifacts

MAY:
- Review the SSM Agent log on the instance for download or extraction errors
- Compare manifest against a known working package

## Phase 2 — Remediate

MUST:
- Fix manifest.json schema errors (schemaVersion must be "2.0", files must reference valid S3 keys)
- Upload missing artifacts to S3 in the correct region
- Ensure install/uninstall scripts have correct shebang lines and permissions

SHOULD:
- Use the SSM console Advanced package creation wizard for validation
- Test the package on a single instance before creating an association

MAY:
- Implement CI/CD pipeline for package building and validation
- Use checksums in manifest to detect artifact corruption

## Common Issues

- symptoms: "CreateDocument fails with InvalidDocumentContent"
  diagnosis: "manifest.json does not conform to the Distributor package schema."
  resolution: "Validate manifest.json: ensure schemaVersion is '2.0', files section references valid attachments, and platform entries are correct."

- symptoms: "Installation fails with 'attachment not found' error"
  diagnosis: "S3 artifacts referenced in manifest do not exist or are inaccessible."
  resolution: "Upload artifacts to the correct S3 path. Verify bucket region and IAM permissions."

## Safety Ratings

```
safety_ratings:
  - "describe-document: GREEN — read-only package metadata"
  - "s3 ls: GREEN — read-only S3 listing"
  - "CreateDocument/UpdateDocument: YELLOW — creates or updates package"
  - "send-command for testing: YELLOW — installs software on instances"
```

## Escalation Conditions

- Custom package creation fails with undocumented errors after manifest validation
- Package artifacts are too large for reliable distribution (>1 GB)
- Custom install scripts require elevated privileges not available on target instances
- Package must integrate with third-party software with complex dependencies

## Data Sensitivity

- `aws ssm describe-document`: LOW — returns package metadata
- `aws s3 ls`: LOW — returns object keys and sizes
- manifest.json: LOW — package structure metadata
- Install/uninstall scripts: MEDIUM — may contain system configuration logic and paths

## Prohibited Actions

- NEVER embed credentials, API keys, or secrets in package artifacts or scripts
- NEVER create packages with install scripts that run as root without explicit justification
- NEVER skip manifest validation before creating or updating a package
- NEVER use hardcoded S3 paths that may change between environments

## Phase 3 — Rollback

1. If a broken custom package was deployed, uninstall from affected instances
2. Revert to the previous package version: `aws ssm update-document-default-version --name <package-name> --document-version <previous-version>`
3. Remove incorrect S3 artifacts: `aws s3 rm s3://<bucket>/packages/<package-name>/<bad-version>/ --recursive`
4. If the package document is corrupted, delete and recreate only if no associations reference it
5. Document the root cause and update the package build process

## Output Format

```yaml
root_cause: "custom_package — <specific_cause>"
evidence:
  - type: manifest_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix manifest or artifacts"
  long_term: "Implement package CI/CD pipeline"
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
