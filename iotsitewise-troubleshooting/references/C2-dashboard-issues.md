---
title: "C2 — Dashboard Issues"
description: "Diagnose SiteWise Monitor dashboard display problems"
status: active
severity: MEDIUM
triggers:
  - "dashboard not loading"
  - "widget error"
  - "no data in dashboard"
  - "visualization wrong"
owner: devops-agent
objective: "Identify and resolve dashboard display issues"
context: "Dashboard issues occur when: asset not in project, property not visualizable, time range has no data, or widget configuration invalid."
---

## Phase 1 — Triage

MUST:
- Verify asset is associated with the project
- Check property has data in the selected time range
- Verify dashboard definition: `aws iotsitewise describe-dashboard --dashboard-id <id>`
- Check user has viewer or admin role on the project

SHOULD:
- Verify property data type is compatible with widget type
- Check browser console for JavaScript errors
- Test with a simple widget first

MAY:
- Export and review dashboard definition JSON
- Check if property aliases are correctly configured

## Phase 2 — Remediate

MUST:
- Add asset to project if missing
- Fix widget property references
- Ensure data exists in the time range

SHOULD:
- Use appropriate widget types for data types
- Test dashboards with known data before sharing

## Output Format

```yaml
root_cause: "dashboard_issue — <specific_cause>"
evidence:
  - type: dashboard_config
    content: "<dashboard definition>"
severity: MEDIUM
mitigation:
  immediate: "Fix widget configuration or data"
  long_term: "Implement dashboard testing procedures"
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
