---
title: "Z1 — General CodeCatalyst Troubleshooting"
description: "Catch-all runbook for CodeCatalyst issues"
status: active
severity: MEDIUM
triggers: ["CodeCatalyst error", "CodeCatalyst not working", "CodeCatalyst issue"]
owner: devops-agent
objective: "Systematic investigation of unclassified CodeCatalyst issues"
context: "Use when the issue doesn't clearly match a specific runbook."
---

## Phase 1 — Triage

MUST:
- List spaces: `aws codecatalyst list-spaces`
- Check space status: `aws codecatalyst get-space --name <space>`
- List projects: `aws codecatalyst list-projects --space-name <space>`

SHOULD:
- Review CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=codecatalyst.amazonaws.com --max-results 20`
- Check Builder ID authentication status

MAY:
- Check CodeCatalyst service status page

## Phase 2 — Remediate

MUST:
- Identify problem domain and pivot to specific runbook

SHOULD:
- Document findings and investigation path

MAY:
- Set up monitoring for the identified issue type

## Safety Ratings

```
safety_ratings:
  - "list-spaces: GREEN — read-only enumeration"
  - "get-space: GREEN — read-only space details"
  - "list-projects: GREEN — read-only project listing"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
```

## Escalation Conditions

- CodeCatalyst service appears non-functional
- Multiple components failing simultaneously
- CloudTrail shows unauthorized API calls
- Issue cannot be classified into any specific runbook after systematic investigation

## Data Sensitivity

- `aws codecatalyst list-spaces`: LOW — returns space names
- `aws codecatalyst get-space`: LOW — returns space configuration
- `aws codecatalyst list-projects`: LOW — returns project names
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history and principal identities

## Prohibited Actions

- NEVER suggest deleting spaces or projects as a first troubleshooting step
- NEVER make configuration changes during general troubleshooting without identifying root cause
- NEVER dismiss the issue without documenting the investigation path
- NEVER share Builder ID credentials or access tokens

## Phase 3 — Rollback

1. If any configuration changes were made during investigation, revert them
2. Verify space and project status returns to normal
3. Confirm workflows and dev environments are functioning
4. Verify team member access is restored
5. Document all investigation steps and findings

## Common Issues

- symptoms: "General CodeCatalyst error with no specific error message"
  diagnosis: "Issue may span multiple domains — space, project, workflow, or dev environment."
  resolution: "Follow triage steps systematically. Check space → projects → workflows → dev environments → connections."

## Output Format

```yaml
root_cause: "general_codecatalyst — <specific_cause>"
evidence:
  - type: investigation_summary
    content: "<findings>"
severity: MEDIUM
mitigation:
  immediate: "Address identified issue"
  long_term: "Implement comprehensive monitoring"
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
