---
title: "B1 — Document Creation Issues"
description: "Diagnose SSM Automation document creation and validation failures"
status: active
severity: MEDIUM
triggers: ["cannot create document", "document validation failed", "CreateDocument error", "invalid document content"]
owner: devops-agent
objective: "Resolve automation document creation failures"
context: "SSM Automation documents use JSON or YAML schema with specific step types, parameters, and output definitions. Documents must pass schema validation before creation."
---

## Phase 1 — Triage

MUST:
- Attempt to validate the document content: `aws ssm create-document --content file://document.json --name test-doc --document-type Automation --dry-run` (if supported)
- Check existing documents for name conflicts: `aws ssm list-documents --document-filter-list key=Name,value=<document-name>`
- Verify document schema version and format

SHOULD:
- Review the document content for schema compliance
- Check that all step types are valid (aws:runCommand, aws:executeScript, aws:approve, etc.)
- Verify parameter definitions have correct types and constraints

MAY:
- Compare against a known working document template
- Check CloudTrail for CreateDocument error details: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateDocument --max-results 5`

## Phase 2 — Remediate

MUST:
- Fix document schema errors (missing required fields, invalid step types)
- Resolve document name conflicts by using a unique name
- Ensure schemaVersion is "0.3" for Automation documents

SHOULD:
- Validate all parameter references in steps match defined parameters
- Ensure output definitions reference valid step outputs
- Add description fields for parameters and steps

MAY:
- Use AWS-provided document templates as starting points
- Implement document creation via CloudFormation for version control

## Safety Ratings

```
safety_ratings:
  - "list-documents: GREEN — read-only document listing"
  - "create-document: YELLOW — creates new SSM document resource"
  - "delete-document: RED — permanently removes document and all versions"
  - "describe-document: GREEN — read-only document metadata"
```

## Escalation Conditions

- Document creation fails with internal service errors after fixing all schema issues
- Document size exceeds the 64KB limit and cannot be reduced
- Custom step types or advanced features produce undocumented errors
- Document creation blocked by SSM document quota limits

## Data Sensitivity

- `aws ssm list-documents`: LOW — returns document names and metadata
- `aws ssm create-document`: MEDIUM — document content may contain operational logic
- `aws ssm get-document`: MEDIUM — returns full document content
- `aws cloudtrail lookup-events`: MEDIUM — reveals IAM principals and actions

## Prohibited Actions

- NEVER delete an existing document to resolve name conflicts without verifying no active executions reference it
- NEVER create documents with hardcoded credentials or secrets in parameters
- NEVER bypass schema validation by modifying document format
- NEVER create documents with overly permissive IAM actions (e.g., iam:*)

## Phase 3 — Rollback

1. If a document was created with errors, delete it: `aws ssm delete-document --name <name>` (only if no executions reference it)
2. If a document replaced an existing one incorrectly, restore from version control
3. Verify no automation schedules or maintenance windows reference the problematic document
4. Re-create the document with corrected content after validation
5. Update any references to the document name if it was renamed

## Common Issues

- symptoms: "CreateDocument fails with DocumentAlreadyExists"
  diagnosis: "A document with the same name already exists in the account and region."
  resolution: "Use a different name, or update the existing document with update-document instead."

- symptoms: "CreateDocument fails with InvalidDocumentContent"
  diagnosis: "The document JSON/YAML has schema errors — missing fields, invalid step types, or malformed references."
  resolution: "Validate against the Automation document schema. Check schemaVersion is 0.3, all steps have valid types, and parameters are correctly defined."

- symptoms: "Document created but execution fails immediately"
  diagnosis: "Document passes schema validation but has logical errors in step definitions or parameter references."
  resolution: "Test with a dry run. Verify all {{parameter}} and {{stepName.outputKey}} references resolve correctly."

## Output Format

```yaml
root_cause: "document_creation — <specific_cause>"
evidence:
  - type: creation_error
    content: "<error details from CreateDocument>"
severity: MEDIUM
mitigation:
  immediate: "Fix document content"
  long_term: "Implement document validation in CI/CD pipeline"
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
