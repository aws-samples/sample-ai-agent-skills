---
title: "F1 — Web Experience Setup Issues"
description: "Diagnose web experience creation and access problems"
status: active
severity: MEDIUM
triggers:
  - "web experience not loading"
  - "cannot access chat interface"
  - "web experience creation failed"
owner: devops-agent
objective: "Resolve web experience setup issues"
context: "Web experience provides a chat interface. Requires IAM Identity Center user assignment."
---

## Phase 1 — Triage

MUST:
- Check web experience: `aws qbusiness get-web-experience --application-id <app-id> --web-experience-id <we-id>`
- Verify web experience status
- Check IAM Identity Center user assignments
- Test web experience URL

SHOULD:
- Verify user roles (admin, user)
- Check browser compatibility
- Verify network access to endpoint

## Phase 2 — Remediate

MUST:
- Fix web experience configuration
- Assign users in Identity Center
- Verify endpoint accessibility

SHOULD:
- Test with multiple users
- Document access requirements

## Common Issues

- symptoms: "Web experience URL returns access denied"
  diagnosis: "User not assigned in IAM Identity Center."
  resolution: "Assign user to Q Business application in Identity Center."

## Output Format

```yaml
root_cause: "web_experience — <specific_cause>"
evidence:
  - type: experience_status
    content: "<web experience configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix access configuration"
  long_term: "Automate user provisioning"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: MEDIUM — web experience provides user access to enterprise data
- Service Disruption Risk: MEDIUM — setup issues prevent user access
- Compliance Impact: HIGH — web experience must enforce authentication and access controls

## Escalation Conditions
- Web experience setup fails
- Authentication integration fails
- Web experience inaccessible to authorized users
- Web experience accessible to unauthorized users
- Setup changes break existing user access

## Data Sensitivity
- Web experience configurations — INTERNAL
- Authentication settings — CONFIDENTIAL
- User access configurations — CONFIDENTIAL
- Web experience URLs — INTERNAL
- Never expose web experience admin credentials

## Prohibited Actions
- Do NOT deploy web experience without authentication
- Do NOT expose web experience URLs publicly without access controls
- Do NOT bypass authentication requirements
- Do NOT ignore web experience access issues
- Do NOT share web experience admin credentials
- Do NOT disable access logging

## Phase 3 — Rollback
- Restore previous web experience configurations
- Revert authentication changes
- Restore user access settings
- Verify web experience accessibility after changes
- Document setup issues and resolutions
- Update web experience management procedures

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

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
