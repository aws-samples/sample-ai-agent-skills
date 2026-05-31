---
title: "Z1 — General Personalize Troubleshooting"
description: "Catch-all runbook for Personalize issues"
status: active
severity: MEDIUM
triggers: ["personalize error", "personalize not working", "recommendation issue"]
owner: devops-agent
objective: "Systematically diagnose unspecified Personalize issues"
context: "Use this runbook when the issue doesn't clearly map to a specific runbook."
---

## Phase 1 — Triage

MUST:
- List dataset groups: `aws personalize list-dataset-groups`
- Check solutions: `aws personalize list-solutions`
- Check campaigns: `aws personalize list-campaigns`
- Review CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=personalize.amazonaws.com --max-results 20`

SHOULD:
- Check dataset import status
- Verify solution version status
- Review campaign status

MAY:
- Check service quotas
- Review IAM permissions

## Phase 2 — Remediate

MUST:
- Route to specific runbook once problem domain identified

SHOULD:
- Check pipeline: data → schema → dataset → import → solution → version → campaign

MAY:
- Engage AWS Support if issue persists

## Common Issues

- symptoms: "General Personalize errors"
  diagnosis: "Follow the pipeline: data → training → deployment → inference."
  resolution: "Check each stage. Route to specific runbook."

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: pipeline_status
    content: "<pipeline stage statuses>"
severity: MEDIUM
mitigation:
  immediate: "Route to specific runbook"
  long_term: "Implement end-to-end pipeline monitoring"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: MEDIUM — troubleshooting may involve inspecting user data configurations
- Service Disruption Risk: LOW — diagnostic operations are read-only
- Compliance Impact: LOW — standard troubleshooting procedures

## Escalation Conditions
- Issue cannot be resolved using standard troubleshooting procedures
- Personalize service becomes unavailable
- Multiple unrelated issues occur simultaneously
- AWS Health Dashboard shows Personalize service event
- Issue requires AWS Support engagement

## Data Sensitivity
- Service configurations — INTERNAL
- CloudWatch metrics — INTERNAL
- Error messages — INTERNAL
- User data references — RESTRICTED
- Never expose user behavioral data during troubleshooting

## Prohibited Actions
- Do NOT inspect individual user data without privacy safeguards
- Do NOT make changes to production configurations without approval
- Do NOT share service details in public forums without sanitizing
- Do NOT bypass established escalation procedures
- Do NOT attempt fixes outside Personalize service boundaries
- Do NOT ignore recurring issues

## Phase 3 — Rollback
- Revert any diagnostic changes made during troubleshooting
- Restore service to last known good configuration
- Document all troubleshooting steps and outcomes
- Verify service health after troubleshooting
- Update runbooks with new findings
- Close out temporary access granted for troubleshooting

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
