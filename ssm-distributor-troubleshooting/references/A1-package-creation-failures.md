---
title: "A1 — Package Creation Failures"
description: "Diagnose SSM Distributor package creation failures"
status: active
severity: HIGH
triggers: ["cannot create package", "package creation failed", "create-document error"]
owner: devops-agent
objective: "Resolve Distributor package creation issues"
context: "Distributor packages are SSM Documents of type Package with a manifest.json defining platform-specific installers."
---

## Phase 1 — Triage

MUST:
- Check existing packages: `aws ssm list-documents --document-filter-list key=DocumentType,value=Package --query 'DocumentIdentifiers[*].{Name:Name,Owner:Owner}'`
- Verify IAM permissions for ssm:CreateDocument
- Validate manifest.json structure: ensure it contains files, version, and platform entries

SHOULD:
- Check if a package with the same name already exists in the region
- Verify S3 bucket accessibility for custom package artifacts: `aws s3 ls s3://<bucket>/packages/`

MAY:
- Review CloudTrail for CreateDocument errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateDocument --max-results 10`

## Phase 2 — Remediate

MUST:
- Fix manifest.json schema errors (missing required fields, invalid platform names)
- Ensure IAM role has ssm:CreateDocument and ssm:AddTagsToResource permissions
- Verify S3 artifacts exist and are accessible from the target region

SHOULD:
- Use the SSM console to validate package structure before CLI creation
- Tag packages with environment and owner metadata

MAY:
- Automate package creation with CloudFormation or Terraform

## Common Issues

- symptoms: "CreateDocument returns InvalidDocumentContent for Package type"
  diagnosis: "manifest.json has invalid structure or missing required fields."
  resolution: "Validate manifest.json against the Distributor schema. Ensure files, version, and platform sections are present."

- symptoms: "CreateDocument returns DocumentAlreadyExists"
  diagnosis: "A package with the same name already exists in this region."
  resolution: "Use a unique name or add a new version to the existing package with aws ssm update-document."

## Safety Ratings

```
safety_ratings:
  - "list-documents: GREEN — read-only enumeration of existing packages"
  - "CreateDocument: YELLOW — creates a new SSM document resource"
  - "s3 ls: GREEN — read-only listing of S3 objects"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
```

## Escalation Conditions

- Package creation fails with internal service errors after fixing manifest and permissions
- Package creation blocked by SCP or Organizations policy
- Custom package requires cross-account S3 access that cannot be configured
- Manifest validation passes but package creation still fails

## Data Sensitivity

- `aws ssm list-documents`: LOW — returns document metadata only
- `aws s3 ls`: LOW — returns object keys and sizes
- `aws cloudtrail lookup-events`: MEDIUM — may reveal IAM principal identities and actions
- manifest.json content: LOW — contains package structure metadata

## Prohibited Actions

- NEVER delete an existing package to recreate it without verifying no associations reference it
- NEVER create packages with overly permissive IAM policies
- NEVER store sensitive data (credentials, keys) in package artifacts or scripts
- NEVER skip manifest validation before creating a package

## Phase 3 — Rollback

1. If a package was created with incorrect content, delete it only if no associations reference it: `aws ssm delete-document --name <package-name>`
2. If S3 artifacts were uploaded to the wrong bucket, remove them: `aws s3 rm s3://<bucket>/packages/<package-name>/ --recursive`
3. If IAM permissions were broadened for troubleshooting, revert to least-privilege
4. Verify no orphaned associations exist pointing to deleted packages
5. Document the rollback action and notify the operations team

## Output Format

```yaml
root_cause: "package_creation — <specific_cause>"
evidence:
  - type: creation_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix manifest or permissions"
  long_term: "Automate package creation via IaC"
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
