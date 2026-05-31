---
title: "Z1 — General Evidently Troubleshooting"
description: "Catch-all runbook for Evidently issues"
status: active
severity: MEDIUM
triggers: ["Evidently error", "evidently not working", "feature flag issue"]
owner: devops-agent
objective: "Systematic investigation of unclassified Evidently issues"
context: "Use when the issue doesn't clearly match a specific runbook."
---

## Phase 1 — Triage

MUST:
- List projects: `aws evidently list-projects`
- Get project details: `aws evidently get-project --project <name>`
- List features: `aws evidently list-features --project <name>`

SHOULD:
- Review CloudTrail for Evidently API errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=evidently.amazonaws.com --max-results 20`
- Check for active launches and experiments
- Verify IAM permissions for the caller

MAY:
- Check AWS Health Dashboard for Evidently service issues
- Review project data delivery configuration

## Phase 2 — Remediate

MUST:
- Identify problem domain and pivot to specific runbook
- Document initial findings and investigation path

SHOULD:
- Verify project is in AVAILABLE status
- Check if the issue is feature-related, launch-related, experiment-related, or configuration-related

MAY:
- Set up monitoring for Evidently API errors
- Create CloudWatch alarms on Evidently metrics

## Common Issues

- symptoms: "Evidently features not returning expected variations"
  diagnosis: "Multiple factors can affect evaluation: overrides, launches, experiments, default variation."
  resolution: "Check in order: 1) Entity overrides, 2) Active launches, 3) Active experiments, 4) Default variation. Use evaluate-feature to test."

## Safety Ratings

```
safety_ratings:
  - "list-projects: GREEN — read-only enumeration"
  - "get-project: GREEN — read-only project details"
  - "list-features: GREEN — read-only feature listing"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
```

## Escalation Conditions

- Evidently service appears non-functional across multiple projects
- Feature evaluations returning errors for all entities
- CloudTrail shows unauthorized API calls to Evidently
- Issue cannot be classified into any specific runbook after systematic investigation

## Data Sensitivity

- `aws evidently list-projects`: LOW — returns project metadata
- `aws evidently get-project`: LOW — returns project configuration
- `aws evidently list-features`: LOW — returns feature metadata
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history and principal identities

## Prohibited Actions

- NEVER delete a project during general troubleshooting
- NEVER stop active launches or experiments without first identifying the root cause
- NEVER dismiss the issue without documenting the investigation path
- NEVER make configuration changes during general troubleshooting without identifying root cause first

## Phase 3 — Rollback

1. If any configuration changes were made during investigation, revert them
2. Restore project, feature, launch, or experiment settings if modified
3. Verify all features are evaluating correctly before closing
4. Document all investigation steps and findings
5. Confirm active launches and experiments are unaffected

## Output Format

```yaml
root_cause: "general_evidently — <specific_cause>"
evidence:
  - type: investigation_summary
    content: "<findings>"
severity: MEDIUM
mitigation:
  immediate: "Address identified issue"
  long_term: "Implement monitoring and alerting for Evidently health"
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
