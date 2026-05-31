---
title: "A2 — Directory Integration"
description: "Diagnose WorkDocs directory integration issues"
status: active
severity: HIGH
triggers: ["directory not syncing", "AD integration failed", "users not appearing", "directory connection"]
owner: devops-agent
objective: "Resolve directory integration issues including AD sync, user provisioning, and authentication"
context: "WorkDocs relies on Directory Service for user authentication. AD Connector connects to on-premises AD. Managed Microsoft AD is standalone."
---

## Phase 1 — Triage

MUST:
- Check directory status: `aws ds describe-directories --directory-ids <directory-id>`
- Verify directory connectivity (for AD Connector): check VPN/Direct Connect status
- List WorkDocs users to check sync status: `aws workdocs describe-users --organization-id <org-id> --query 'Users[*].{Username:Username,Status:Status}'`

SHOULD:
- Check directory event logs: `aws ds describe-event-topics --directory-id <directory-id>`
- Verify DNS resolution for the directory domain
- Check security group rules for directory communication ports (389, 636, 88, 53)

MAY:
- Review CloudTrail for directory-related errors
- Check VPC flow logs for blocked traffic to directory endpoints

## Phase 2 — Remediate

MUST:
- Fix network connectivity between AD Connector and on-premises AD
- Ensure security groups allow required ports for directory communication
- Verify DNS settings resolve the directory domain correctly

SHOULD:
- Re-sync users if directory sync is stale
- Update AD Connector service account credentials if expired

MAY:
- Set up CloudWatch alarms for directory health metrics
- Configure SNS notifications for directory status changes

## Safety Ratings

```
safety_ratings:
  - "ds describe-directories: GREEN — read-only directory inspection"
  - "workdocs describe-users: GREEN — read-only user listing"
  - "ds describe-event-topics: GREEN — read-only event topic listing"
  - "ds reset-user-password: RED — changes user authentication credentials"
```

## Escalation Conditions

- AD Connector cannot reach on-premises AD despite correct network configuration
- Directory status is Impaired or Failed and cannot be restored
- User sync has been stale for more than 24 hours
- Authentication failures affect all users simultaneously

## Data Sensitivity

- `aws ds describe-directories`: MEDIUM — reveals directory configuration and network details
- `aws workdocs describe-users`: MEDIUM — reveals user identities and email addresses
- AD Connector service account: HIGH — provides access to on-premises Active Directory
- VPC flow logs: MEDIUM — reveals network traffic patterns

## Prohibited Actions

- NEVER reset the AD Connector service account password without coordinating with the on-premises AD team
- NEVER modify directory security groups without understanding the impact on all dependent services
- NEVER delete or recreate the directory to fix sync issues
- NEVER expose directory credentials in troubleshooting logs or outputs

## Phase 3 — Rollback

1. If security group rules were modified, revert to original rules
2. If DNS settings were changed, restore original DNS configuration
3. If AD Connector settings were modified, restore original service account configuration
4. Verify all users can authenticate after any directory changes
5. Document directory integration configuration for the operations team

## Common Issues

- symptoms: "New AD users not appearing in WorkDocs"
  diagnosis: "Directory sync has not completed or user is not in the synced OU."
  resolution: "Wait for sync cycle (up to 5 minutes for Simple AD, varies for AD Connector). Verify user is in the correct OU."

- symptoms: "Users cannot authenticate to WorkDocs"
  diagnosis: "AD Connector has lost connectivity to on-premises AD."
  resolution: "Check VPN/Direct Connect status. Verify security groups allow ports 389, 636, 88, 53 to on-premises AD."

## Output Format

```yaml
root_cause: "directory_integration — <specific_cause>"
evidence:
  - type: directory_issue
    content: "<directory status and connectivity details>"
severity: HIGH
mitigation:
  immediate: "Fix directory connectivity"
  long_term: "Monitor directory health with CloudWatch"
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
