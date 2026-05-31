---
title: "Z1 — General WorkMail Troubleshooting"
description: "Catch-all runbook for WorkMail issues"
status: active
severity: MEDIUM
triggers: ["WorkMail error", "WorkMail not working", "email service issue"]
owner: devops-agent
objective: "Systematic investigation of unclassified WorkMail issues"
context: "Use when the issue doesn't clearly match a specific runbook."
---

## Phase 1 — Triage

MUST:
- Verify organization is active: `aws workmail describe-organization --organization-id <org-id>`
- List users and check status: `aws workmail list-users --organization-id <org-id> --query 'Users[*].{Name:Name,State:State}'`
- Check domain status: `aws workmail list-mail-domains --organization-id <org-id>`
- Review CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=workmail.amazonaws.com --max-results 20`

SHOULD:
- Check directory health: `aws ds describe-directories`
- Verify SES sending status: `aws ses get-account`
- Check AWS Service Health Dashboard for WorkMail

MAY:
- Check access control rules: `aws workmail list-access-control-rules --organization-id <org-id>`
- Review CloudWatch metrics for anomalies

## Phase 2 — Remediate

MUST:
- Identify problem domain and pivot to specific runbook (A1-F1)
- Document findings from triage

SHOULD:
- Categorize the issue: organization, email flow, client, mailbox, migration, or security
- Collect relevant IDs (organization ID, user ID, domain) for the specific runbook

MAY:
- Set up monitoring for the identified issue category
- Create a test user to isolate the problem

## Safety Ratings

```
safety_ratings:
  - "workmail describe-organization: GREEN — read-only organization inspection"
  - "workmail list-users: GREEN — read-only user listing"
  - "workmail list-mail-domains: GREEN — read-only domain listing"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
```

## Escalation Conditions

- WorkMail service appears non-functional in the region
- Directory Service is impaired affecting all WorkMail operations
- SES account is suspended affecting outbound email
- Issue cannot be classified into any specific runbook after systematic investigation

## Data Sensitivity

- `aws workmail describe-organization`: LOW — returns organization metadata
- `aws workmail list-users`: MEDIUM — reveals user identities
- `aws workmail list-mail-domains`: LOW — reveals configured domains
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history

## Prohibited Actions

- NEVER delete the organization during general troubleshooting
- NEVER deregister users during investigation
- NEVER make configuration changes without first identifying the root cause
- NEVER dismiss the issue without documenting the investigation path

## Phase 3 — Rollback

1. If any configuration changes were made during investigation, revert them
2. Restore organization settings if they were modified during troubleshooting
3. If test users were created, deregister them after investigation
4. Document all investigation steps and findings for the team
5. Verify WorkMail is functioning normally before closing the investigation

## Common Issues

- symptoms: "All WorkMail operations return errors"
  diagnosis: "Organization may be in an error state or directory is impaired."
  resolution: "Check organization state and directory health. If both are healthy, check AWS Service Health Dashboard."

- symptoms: "Cannot determine which runbook applies"
  diagnosis: "Issue spans multiple domains or is a new failure mode."
  resolution: "Start with organization status, then domain verification, then user status, then email flow. Follow error messages."

## Output Format

```yaml
root_cause: "general_workmail — <specific_cause>"
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
