---
title: "A2 — Project Configuration"
description: "Diagnose Evidently project configuration issues"
status: active
severity: MEDIUM
triggers: ["project settings wrong", "update project failed", "data delivery not configured"]
owner: devops-agent
objective: "Resolve project configuration and update issues"
context: "Project configuration includes data delivery settings, description, and tags. Data delivery is configured at the project level."
---

## Phase 1 — Triage

MUST:
- Get project details: `aws evidently get-project --project <name>`
- Check data delivery configuration: look for dataDelivery field in project details
- Verify project status is AVAILABLE

SHOULD:
- Check if data delivery is configured for S3 or CloudWatch Logs
- Review project tags for organizational compliance
- Verify the project has features, launches, or experiments as expected

MAY:
- Compare current configuration against IaC templates
- Review recent update-project calls in CloudTrail

## Phase 2 — Remediate

MUST:
- Configure data delivery if needed: `aws evidently update-project-data-delivery --project <name> --s3-destination '{"bucket":"<bucket>","prefix":"<prefix>"}'`
- Fix project settings as needed: `aws evidently update-project --project <name> --description "<description>"`

SHOULD:
- Verify data delivery is working by checking the destination (S3 bucket or log group)
- Update project tags for cost allocation

MAY:
- Set up monitoring for data delivery health
- Configure project via IaC for consistency

## Common Issues

- symptoms: "No evaluation data appearing in S3 despite active features"
  diagnosis: "Data delivery may not be configured at the project level."
  resolution: "Check project data delivery: `aws evidently get-project --project <name>`. Configure if missing: `aws evidently update-project-data-delivery --project <name> --s3-destination '{\"bucket\":\"<bucket>\",\"prefix\":\"<prefix>\"}'`"

## Safety Ratings

```
safety_ratings:
  - "get-project: GREEN — read-only project details"
  - "update-project: YELLOW — modifies project settings"
  - "update-project-data-delivery: YELLOW — changes data export configuration"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
```

## Escalation Conditions

- Project stuck in non-AVAILABLE status despite correct configuration
- Data delivery configured but no data appearing after 1 hour
- Configuration changes not taking effect after update
- Project configuration drift detected between IaC and actual state

## Data Sensitivity

- `aws evidently get-project`: LOW — returns project configuration metadata
- `aws evidently update-project-data-delivery`: MEDIUM — modifies data export destination
- Data delivery destination: MEDIUM — evaluation data may contain entity identifiers

## Prohibited Actions

- NEVER disable data delivery on a project that feeds downstream analytics
- NEVER modify project configuration without documenting previous values
- NEVER delete a project to fix configuration issues
- NEVER change data delivery destination without verifying the new destination is accessible

## Phase 3 — Rollback

1. Revert data delivery changes: `aws evidently update-project-data-delivery --project <name>` with previous configuration
2. Restore project description and tags to previous values
3. Verify data delivery resumes to the correct destination
4. Document all configuration changes and rollback actions
5. Confirm features, launches, and experiments are unaffected

## Output Format

```yaml
root_cause: "project_configuration — <specific_cause>"
evidence:
  - type: configuration_issue
    content: "<configuration details>"
severity: MEDIUM
mitigation:
  immediate: "Fix project configuration"
  long_term: "Manage project configuration via IaC"
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
