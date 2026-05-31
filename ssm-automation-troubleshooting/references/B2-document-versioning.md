---
title: "B2 — Document Versioning"
description: "Diagnose SSM Automation document versioning and default version issues"
status: active
severity: MEDIUM
triggers: ["wrong document version", "version mismatch", "$DEFAULT not updated", "document version conflict"]
owner: devops-agent
objective: "Resolve document versioning issues ensuring correct version is used for executions"
context: "SSM documents support multiple versions. $LATEST is the newest version, $DEFAULT is what executions use by default. These are independent — updating a document does NOT change $DEFAULT."
---

## Phase 1 — Triage

MUST:
- List document versions: `aws ssm list-document-versions --name <document-name>`
- Check current default version: `aws ssm describe-document --name <document-name> --query 'Document.{DefaultVersion:DefaultVersion,LatestVersion:LatestVersion}'`
- Compare $DEFAULT and $LATEST version numbers

SHOULD:
- Get content of the default version: `aws ssm get-document --name <document-name> --document-version '$DEFAULT'`
- Get content of the latest version: `aws ssm get-document --name <document-name> --document-version '$LATEST'`
- Check recent executions to see which version they used

MAY:
- Review CloudTrail for UpdateDocument and UpdateDocumentDefaultVersion events
- Compare version contents to identify differences

## Phase 2 — Remediate

MUST:
- Update default version if needed: `aws ssm update-document-default-version --name <document-name> --document-version <version-number>`
- Verify executions use the intended version after the change

SHOULD:
- Establish a versioning workflow (update → test → set default)
- Document version changes and their purposes
- Use specific version numbers in critical automation triggers instead of $DEFAULT

MAY:
- Implement version management via CloudFormation or Terraform
- Set up notifications for document version changes

## Safety Ratings

```
safety_ratings:
  - "list-document-versions: GREEN — read-only version listing"
  - "describe-document: GREEN — read-only document metadata"
  - "update-document-default-version: YELLOW — changes which version executions use by default"
  - "update-document: YELLOW — creates new document version"
```

## Escalation Conditions

- Default version change affects production automation pipelines
- Version rollback needed but previous version has known issues
- Multiple teams depend on different document versions
- Version limit (1000) approaching for a frequently updated document

## Data Sensitivity

- `aws ssm list-document-versions`: LOW — returns version metadata only
- `aws ssm get-document`: MEDIUM — returns full document content with logic
- `aws ssm describe-document`: LOW — returns document metadata
- `aws cloudtrail lookup-events`: MEDIUM — reveals who changed versions and when

## Prohibited Actions

- NEVER change the default version of a production document without testing the new version first
- NEVER delete old document versions that may be referenced by scheduled automations
- NEVER assume $LATEST is $DEFAULT — always verify explicitly
- NEVER update documents during active execution windows without coordination

## Phase 3 — Rollback

1. Revert default version: `aws ssm update-document-default-version --name <document-name> --document-version <previous-version>`
2. Verify the rollback by checking: `aws ssm describe-document --name <document-name>`
3. Notify teams that depend on the document about the version change
4. Test a non-production execution with the reverted version
5. Document the version change and reason for rollback

## Common Issues

- symptoms: "Automation uses old logic despite document being updated"
  diagnosis: "$DEFAULT version was not updated after the document was modified."
  resolution: "Update default version: aws ssm update-document-default-version --name <name> --document-version <latest-version-number>."

- symptoms: "Different executions use different document versions"
  diagnosis: "Some executions specify a version explicitly while others use $DEFAULT."
  resolution: "Standardize version usage. Either always use $DEFAULT or always specify explicit versions."

- symptoms: "Cannot update document — DocumentVersionLimitExceeded"
  diagnosis: "The document has reached the maximum number of versions (1000)."
  resolution: "Delete unused old versions to make room. Consider creating a new document if extensive cleanup is needed."

## Output Format

```yaml
root_cause: "document_versioning — <specific_cause>"
evidence:
  - type: version_mismatch
    content: "<DEFAULT vs LATEST version details>"
severity: MEDIUM
mitigation:
  immediate: "Align document version"
  long_term: "Implement version management workflow"
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
