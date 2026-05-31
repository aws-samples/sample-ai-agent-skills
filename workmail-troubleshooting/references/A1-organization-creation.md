---
title: "A1 — Organization Creation"
description: "Diagnose WorkMail organization creation failures"
status: active
severity: HIGH
triggers: ["cannot create organization", "organization creation failed", "WorkMail setup error"]
owner: devops-agent
objective: "Resolve WorkMail organization creation issues including directory, region, and configuration problems"
context: "WorkMail organizations require a directory (auto-created or existing). Each organization is regional."
---

## Phase 1 — Triage

MUST:
- List existing organizations: `aws workmail list-organizations --query 'OrganizationSummaries[*].{Id:OrganizationId,Alias:Alias,State:State}'`
- Check if the alias is already in use (aliases must be globally unique)
- Verify IAM permissions for workmail:CreateOrganization
- Check if a directory exists or needs to be created: `aws ds describe-directories`

SHOULD:
- Verify the region supports WorkMail
- Check if the organization limit has been reached in the account
- Verify KMS key access if using customer-managed encryption

MAY:
- Review CloudTrail for CreateOrganization errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateOrganization --max-results 10`

## Phase 2 — Remediate

MUST:
- Choose a unique alias for the organization
- Ensure directory is available (create one or use existing)
- Fix IAM permissions for organization creation

SHOULD:
- Configure the default domain during setup
- Set up the admin user account

MAY:
- Use an existing Managed Microsoft AD for integration with other services
- Configure customer-managed KMS key for encryption

## Safety Ratings

```
safety_ratings:
  - "workmail list-organizations: GREEN — read-only organization listing"
  - "ds describe-directories: GREEN — read-only directory listing"
  - "workmail CreateOrganization: YELLOW — creates organization with cost implications"
  - "ds create-directory: RED — creates directory resource with ongoing costs"
```

## Escalation Conditions

- Organization creation fails with internal service errors after all prerequisites are met
- Directory creation fails due to VPC or subnet issues
- Organization alias conflicts cannot be resolved
- KMS key access issues prevent organization creation

## Data Sensitivity

- `aws workmail list-organizations`: LOW — returns organization metadata
- `aws ds describe-directories`: MEDIUM — reveals directory configuration
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history

## Prohibited Actions

- NEVER delete an existing organization to reuse its alias
- NEVER create organizations in unsupported regions
- NEVER use the root account for organization creation
- NEVER share organization admin credentials in logs

## Phase 3 — Rollback

1. If an organization was created in error, delete it: `aws workmail delete-organization --organization-id <id> --delete-directory` (WARNING: destroys all data)
2. If a directory was created separately, delete it only after confirming no services depend on it
3. If KMS key was created for the organization, schedule key deletion if not needed
4. Revert any IAM policy changes made during troubleshooting
5. Document the creation attempt and failure for future reference

## Common Issues

- symptoms: "CreateOrganization fails with 'Alias already in use'"
  diagnosis: "The organization alias is globally unique and already taken."
  resolution: "Choose a different alias. Aliases cannot be reused even after organization deletion."

- symptoms: "Organization stuck in 'Creating' state"
  diagnosis: "Directory creation or configuration is taking longer than expected."
  resolution: "Wait up to 30 minutes. If still creating, check directory status. Contact AWS Support if persistent."

## Output Format

```yaml
root_cause: "organization_creation — <specific_cause>"
evidence:
  - type: creation_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix creation prerequisites"
  long_term: "Use IaC for consistent organization provisioning"
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
