---
title: "Z1 — General WorkDocs Troubleshooting"
description: "Catch-all runbook for WorkDocs issues"
status: active
severity: MEDIUM
triggers: ["WorkDocs error", "WorkDocs not working", "document service issue"]
owner: devops-agent
objective: "Systematic investigation of unclassified WorkDocs issues"
context: "Use when the issue doesn't clearly match a specific runbook."
---

## Phase 1 — Triage

MUST:
- Verify WorkDocs site is active by listing users: `aws workdocs describe-users --organization-id <org-id> --limit 5`
- Check directory health: `aws ds describe-directories --query 'DirectoryDescriptions[*].{Id:DirectoryId,Status:Stage}'`
- Review recent activity: `aws workdocs describe-activities --organization-id <org-id> --limit 20`
- Check CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=workdocs.amazonaws.com --max-results 20`

SHOULD:
- Verify IAM permissions for the caller
- Check AWS Service Health Dashboard for WorkDocs
- Test basic operations (list users, describe folder contents)

MAY:
- Check CloudWatch metrics for anomalies
- Verify network connectivity to WorkDocs endpoints

## Phase 2 — Remediate

MUST:
- Identify problem domain and pivot to specific runbook (A1-F1)
- Document findings from triage

SHOULD:
- Categorize the issue: site, content, sharing, user, migration, storage, or API
- Collect relevant IDs (organization ID, user ID, document ID) for the specific runbook

MAY:
- Set up monitoring for the identified issue category
- Create a test user to isolate the problem

## Safety Ratings

```
safety_ratings:
  - "workdocs describe-users: GREEN — read-only user listing"
  - "ds describe-directories: GREEN — read-only directory inspection"
  - "workdocs describe-activities: GREEN — read-only activity feed"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
```

## Escalation Conditions

- WorkDocs service appears non-functional in the region
- Directory Service is impaired affecting all WorkDocs operations
- CloudTrail shows unauthorized API calls to WorkDocs
- Issue cannot be classified into any specific runbook after systematic investigation

## Data Sensitivity

- `aws workdocs describe-users`: MEDIUM — reveals user identities
- `aws workdocs describe-activities`: MEDIUM — reveals user activity patterns
- `aws ds describe-directories`: MEDIUM — reveals directory configuration
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history

## Prohibited Actions

- NEVER deactivate users during general troubleshooting
- NEVER delete documents or folders during investigation
- NEVER make configuration changes without first identifying the root cause
- NEVER dismiss the issue without documenting the investigation path

## Phase 3 — Rollback

1. If any configuration changes were made during investigation, revert them
2. Restore user settings if they were modified during troubleshooting
3. If test users were created, deactivate them after investigation
4. Document all investigation steps and findings for the team
5. Verify WorkDocs is functioning normally before closing the investigation

## Common Issues

- symptoms: "All WorkDocs operations return errors"
  diagnosis: "Directory Service may be impaired or WorkDocs service issue."
  resolution: "Check directory health first. If directory is healthy, check AWS Service Health Dashboard."

- symptoms: "Cannot determine which runbook applies"
  diagnosis: "Issue spans multiple domains or is a new failure mode."
  resolution: "Start with directory health, then user status, then content access. Follow error messages to the specific domain."

## Output Format

```yaml
root_cause: "general_workdocs — <specific_cause>"
evidence:
  - type: investigation_summary
    content: "<findings>"
severity: MEDIUM
mitigation:
  immediate: "Address identified issue"
  long_term: "Implement monitoring and alerting"
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
