---
title: "A1 — Site Creation Failures"
description: "Diagnose WorkDocs site creation and activation failures"
status: active
severity: HIGH
triggers: ["cannot create site", "site creation failed", "activate WorkDocs failed"]
owner: devops-agent
objective: "Resolve WorkDocs site creation issues including directory requirements and regional constraints"
context: "WorkDocs sites require a Directory Service directory. Each directory can have one WorkDocs site."
---

## Phase 1 — Triage

MUST:
- List existing directories: `aws ds describe-directories --query 'DirectoryDescriptions[*].{Id:DirectoryId,Name:Name,Type:Type,Status:Stage}'`
- Verify directory status is Active
- Check if WorkDocs is already activated on the directory
- Verify the region supports WorkDocs

SHOULD:
- Check directory type compatibility (Simple AD, AD Connector, or Managed Microsoft AD)
- Verify IAM permissions for workdocs:RegisterDirectory and workdocs:ActivateUser
- Check if the directory VPC has proper network configuration

MAY:
- Review CloudTrail for WorkDocs activation events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=workdocs.amazonaws.com --max-results 10`

## Phase 2 — Remediate

MUST:
- Create a Directory Service directory if none exists
- Ensure directory is in Active state before activating WorkDocs
- Activate WorkDocs on the directory through the console or API

SHOULD:
- Configure the admin user during site activation
- Set up initial storage quotas and user policies

MAY:
- Enable multi-factor authentication on the directory
- Configure IP allow lists for site access

## Safety Ratings

```
safety_ratings:
  - "ds describe-directories: GREEN — read-only directory enumeration"
  - "workdocs RegisterDirectory: YELLOW — activates WorkDocs on a directory"
  - "ds create-directory: RED — creates a new directory resource with cost implications"
  - "workdocs ActivateUser: YELLOW — activates a user account"
```

## Escalation Conditions

- Directory creation fails due to VPC or subnet issues
- WorkDocs activation fails with internal service errors after directory is confirmed Active
- Directory type is not supported for WorkDocs in the region
- Organization SCPs block WorkDocs or Directory Service operations

## Data Sensitivity

- `aws ds describe-directories`: MEDIUM — reveals directory names, VPC configuration, and DNS settings
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history and principal identities
- Directory admin credentials: HIGH — provides full administrative access to the WorkDocs site

## Prohibited Actions

- NEVER delete a directory that has an active WorkDocs site — this destroys all content
- NEVER create directories in regions where WorkDocs is not supported
- NEVER share directory admin credentials in troubleshooting logs
- NEVER activate WorkDocs on a production directory without approval

## Phase 3 — Rollback

1. If WorkDocs was activated on the wrong directory, deregister it (note: this may delete content)
2. If a directory was created in error, delete it only after confirming no services depend on it: `aws ds delete-directory --directory-id <id>`
3. If admin user was configured incorrectly, update admin settings through the WorkDocs console
4. Revert any VPC or security group changes made during troubleshooting
5. Document the site creation configuration for future reference

## Common Issues

- symptoms: "Cannot activate WorkDocs — no directories found"
  diagnosis: "No Directory Service directory exists in the region."
  resolution: "Create a Simple AD or Managed Microsoft AD directory first, then activate WorkDocs."

- symptoms: "WorkDocs activation fails with directory in 'Creating' state"
  diagnosis: "Directory is not yet Active."
  resolution: "Wait for directory to reach Active state. This can take 20-45 minutes for Managed Microsoft AD."

## Output Format

```yaml
root_cause: "site_creation — <specific_cause>"
evidence:
  - type: creation_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix directory and activate WorkDocs"
  long_term: "Use IaC for consistent site provisioning"
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
