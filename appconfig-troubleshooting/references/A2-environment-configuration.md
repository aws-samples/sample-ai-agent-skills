---
title: "A2 — Environment Configuration Issues"
description: "Diagnose and resolve AppConfig environment setup and configuration problems"
status: active
severity: MEDIUM
triggers:
  - "environment creation failed"
  - "environment not found"
  - "monitor configuration error"
owner: devops-agent
objective: "Correctly configure AppConfig environments with monitors"
context: "Environments represent deployment targets (dev, staging, prod) and can have CloudWatch alarm monitors for automatic rollback."
---

## Phase 1 — Triage

MUST:
- List environments: `aws appconfig list-environments --application-id <app-id>`
- Get environment details: `aws appconfig get-environment --application-id <app-id> --environment-id <env-id>`
- Check if monitors (CloudWatch alarms) are correctly configured
- Verify IAM permissions for appconfig:CreateEnvironment/UpdateEnvironment

SHOULD:
- Verify CloudWatch alarms referenced as monitors exist and are in the same region
- Check the IAM role used for monitoring has cloudwatch:DescribeAlarms permission

MAY:
- Review CloudTrail for environment-related API calls

## Phase 2 — Remediate

MUST:
- Fix monitor alarm ARNs if they reference non-existent alarms
- Ensure the monitoring IAM role has correct permissions

SHOULD:
- Configure at least one CloudWatch alarm monitor for production environments
- Use descriptive environment names matching your deployment pipeline stages

MAY:
- Add environment-level tags for cost tracking

safety_ratings:
  - "Listing environments: GREEN"
  - "Creating environment: GREEN"
  - "Updating environment monitors: YELLOW"

escalation_conditions:
  - "Production environment monitors are misconfigured"
  - "Environment deletion required for a production application"

data_sensitivity:
  - command: "get-environment"
    sensitivity: LOW
    contains: "Environment name, monitors, state"

prohibited_actions:
  - "NEVER remove monitors from a production environment to fix deployment issues"
  - "NEVER delete a production environment without confirming no active deployments"

## Phase 3 — Rollback

- If environment monitors were changed, restore previous monitor configuration
- Document the original monitor ARNs before making changes

## Common Issues

- symptoms: "Deployment rollback not triggering despite alarm in ALARM state"
  diagnosis: "Monitor IAM role lacks cloudwatch:DescribeAlarms permission or alarm ARN is incorrect"
  resolution: "Verify the monitor role permissions and alarm ARN matches exactly"

- symptoms: "Environment creation fails with validation error"
  diagnosis: "Monitor alarm ARN format is invalid or alarm doesn't exist"
  resolution: "Verify the CloudWatch alarm exists and use the full ARN"

## Output Format

```yaml
root_cause: "<description>"
evidence:
  - type: environment_config
    content: "<environment details>"
severity: MEDIUM
mitigation:
  immediate: "<fix>"
  long_term: "<prevention>"
```
