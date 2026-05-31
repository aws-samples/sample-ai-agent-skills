---
title: "C1 — Portal Access Issues"
description: "Diagnose SiteWise Monitor portal access failures"
status: active
severity: HIGH
triggers:
  - "cannot access portal"
  - "portal login failed"
  - "SSO error"
  - "portal not loading"
owner: devops-agent
objective: "Identify and resolve portal access failures"
context: "Portal access fails when: IAM Identity Center not configured, user not assigned to project, portal role insufficient, or browser compatibility issues."
---

## Phase 1 — Triage

MUST:
- Check portal status: `aws iotsitewise describe-portal --portal-id <id>`
- Verify IAM Identity Center is enabled in the account
- Check portal role ARN and permissions
- Verify user is assigned to a project in the portal

SHOULD:
- Check portal URL is accessible from user's network
- Verify browser compatibility (Chrome, Firefox, Edge)
- Check if portal is in correct region

MAY:
- List portal projects: `aws iotsitewise list-projects --portal-id <id>`
- Check project access assignments

## Phase 2 — Remediate

MUST:
- Enable IAM Identity Center if not configured
- Assign user to portal project with appropriate role
- Fix portal IAM role permissions

SHOULD:
- Use portal admin role for initial setup
- Document portal access procedures

## Output Format

```yaml
root_cause: "portal_access — <specific_cause>"
evidence:
  - type: portal_config
    content: "<portal status and role>"
severity: HIGH
mitigation:
  immediate: "Fix SSO or project assignment"
  long_term: "Automate portal user provisioning"
```

## Safety Ratings
```
safety_ratings:
  - "Triage commands (describe/list/get): GREEN — read-only"
  - "Configuration changes: YELLOW — recoverable"
  - "Delete/terminate operations: RED — irreversible"
```

## Escalation Conditions
- "Resource serves production workload"
- "Fix requires deleting or recreating resources"
- "Fix involves security or encryption changes"
- "Blast radius affects multiple dependent services"

## Data Sensitivity
- HIGH: credentials, PII, security configurations
- MEDIUM: resource configurations, metrics
- LOW: status information, error messages

## Prohibited Actions
- Never delete asset models that have active assets associated with them
- Never modify measurement properties on assets receiving live industrial data
- Never remove gateway configurations without confirming data source migration
- Never alter transform or metric computations on production assets without testing
- Never delete portal or dashboard configurations used by operations teams

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Restore previous asset model configuration if property computations break
- Revert gateway data source settings if industrial data ingestion stops
- Re-enable previous alarm configurations if monitoring coverage is lost

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-instances"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-console-output"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "ssm send-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound security group rules as a fix"
  - "NEVER suggest disabling instance metadata service"
  - "NEVER terminate instances without confirmation"
