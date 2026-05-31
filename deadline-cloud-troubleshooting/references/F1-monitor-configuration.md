---
title: "F1 — Monitor Configuration"
description: "Diagnose Deadline Cloud monitor (web portal) configuration issues"
status: active
severity: MEDIUM
triggers: ["monitor not working", "cannot access portal", "monitor creation failed", "web portal error"]
owner: devops-agent
objective: "Resolve monitor configuration issues for web portal access"
context: "Deadline Cloud monitors provide a web portal for artists and supervisors to view jobs, submit work, and manage queues. Monitors require IAM Identity Center for user authentication."
---

## Phase 1 — Triage

MUST:
- List monitors: `aws deadline list-monitors --query 'monitors[*].{MonitorId:monitorId,DisplayName:displayName,Url:url,IdentityCenterInstanceArn:identityCenterInstanceArn}'`
- Get monitor details: `aws deadline get-monitor --monitor-id <monitor-id>`
- Verify IAM Identity Center is configured and active

SHOULD:
- Check Identity Center instance status
- Verify users are assigned to the monitor
- Test monitor URL accessibility

MAY:
- Review Identity Center application assignments
- Check browser compatibility and network access to monitor URL

## Phase 2 — Remediate

MUST:
- Configure IAM Identity Center if not set up
- Create monitor with Identity Center association: `aws deadline create-monitor --display-name <name> --identity-center-instance-arn <arn> --role-arn <role-arn> --subdomain <subdomain>`
- Assign users in Identity Center to the monitor

SHOULD:
- Configure monitor IAM role with appropriate permissions
- Set up user groups in Identity Center for team management

MAY:
- Customize monitor subdomain for branding
- Configure monitor-level permissions for different user roles

## Safety Ratings

```
safety_ratings:
  - "list-monitors: GREEN — read-only monitor enumeration"
  - "get-monitor: GREEN — read-only monitor details"
  - "create-monitor: YELLOW — creates web portal resource"
  - "delete-monitor: RED — removes web portal access for all users"
  - "Identity Center changes: YELLOW — affects user authentication"
```

## Escalation Conditions

- IAM Identity Center not available in the account requiring organization admin
- Monitor creation fails with Identity Center integration errors
- User authentication issues requiring Identity Center troubleshooting
- Monitor URL not accessible due to network or DNS issues

## Data Sensitivity

- Monitor URL: LOW — web portal endpoint
- Identity Center configuration: MEDIUM — reveals authentication setup
- User assignments: MEDIUM — reveals team structure and access
- Monitor IAM role: MEDIUM — execution role permissions

## Prohibited Actions

- NEVER delete a monitor used by active production teams
- NEVER modify Identity Center configuration without understanding impact on all applications
- NEVER assign admin-level monitor permissions to all users
- NEVER share monitor URLs with unauthenticated external users

## Phase 3 — Rollback

1. If monitor was deleted, recreate with same configuration and reassign users
2. If Identity Center assignment was changed, restore previous user assignments
3. If monitor role was modified, revert IAM role permissions
4. Verify all users can access the monitor after fix

## Common Issues

- symptoms: "Monitor creation fails with Identity Center error"
  diagnosis: "IAM Identity Center not enabled or instance ARN is incorrect."
  resolution: "Enable IAM Identity Center in the account. Verify instance ARN with sso-admin list-instances."

- symptoms: "Users cannot log in to monitor portal"
  diagnosis: "Users not assigned to the monitor in Identity Center."
  resolution: "Assign users or groups to the Deadline Cloud application in Identity Center."

## Output Format

```yaml
root_cause: "monitor_configuration — <specific_cause>"
evidence:
  - type: monitor_detail
    content: "<monitor configuration and Identity Center status>"
severity: MEDIUM
mitigation:
  immediate: "Fix monitor or Identity Center configuration"
  long_term: "Automate user provisioning with Identity Center groups"
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
