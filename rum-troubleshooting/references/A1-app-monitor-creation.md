---
title: "A1 — App Monitor Creation"
description: "Diagnose app monitor creation failures"
status: active
severity: HIGH
triggers: ["cannot create app monitor", "CreateAppMonitor error", "app monitor creation failed"]
owner: devops-agent
objective: "Resolve app monitor creation issues for CloudWatch RUM"
context: "App monitors require a domain, Cognito Identity pool, and appropriate IAM permissions. Each monitor is regional."
---

## Phase 1 — Triage

MUST:
- Check existing app monitors: `aws rum list-app-monitors`
- Verify IAM permissions for rum:CreateAppMonitor
- Confirm the domain name is valid and matches the application's domain

SHOULD:
- Check if a Cognito Identity pool exists or needs to be created
- Verify service quotas for maximum app monitors per account
- Check if an app monitor with the same name already exists

MAY:
- Review CloudTrail for CreateAppMonitor errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=rum.amazonaws.com --max-results 10`

## Phase 2 — Remediate

MUST:
- Fix IAM permissions to include rum:CreateAppMonitor and cognito-identity permissions
- Provide a valid domain name for the application
- Configure Cognito Identity pool for unauthenticated access

SHOULD:
- Set appropriate session sample rate based on traffic volume and budget
- Configure telemetry types (performance, errors, HTTP) during creation
- Enable or disable custom events based on requirements

MAY:
- Enable session replay if needed (increases cost)
- Tag the app monitor for cost allocation
- Configure CW RUM to create the Cognito Identity pool automatically

## Common Issues

- symptoms: "CreateAppMonitor returns AccessDeniedException"
  diagnosis: "IAM permissions missing for rum:CreateAppMonitor or cognito-identity:CreateIdentityPool."
  resolution: "Grant rum:CreateAppMonitor, cognito-identity:CreateIdentityPool, cognito-identity:SetIdentityPoolRoles, and iam:PassRole permissions."

## Safety Ratings

```
safety_ratings:
  - "list-app-monitors: GREEN — read-only enumeration"
  - "CreateAppMonitor: YELLOW — creates app monitor and optionally Cognito resources"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
  - "service-quotas get-service-quota: GREEN — read-only quota check"
```

## Escalation Conditions

- App monitor creation fails with internal errors after fixing all parameters
- Cognito Identity pool creation fails due to account-level restrictions
- Service quota limit reached for app monitors
- Cross-account Cognito configuration required

## Data Sensitivity

- `aws rum list-app-monitors`: LOW — returns app monitor metadata
- `aws cognito-identity create-identity-pool`: MEDIUM — creates authentication resource
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history

## Prohibited Actions

- NEVER create Cognito Identity pools with overly broad permissions
- NEVER set session sample rate to 1.0 on high-traffic applications without cost analysis
- NEVER enable session replay without understanding privacy implications
- NEVER delete existing app monitors to work around creation limits

## Phase 3 — Rollback

1. If an app monitor was created with wrong configuration, update it: `aws rum update-app-monitor --name <name>`
2. If the app monitor should not exist, delete it: `aws rum delete-app-monitor --name <name>`
3. If a Cognito Identity pool was created in error, delete it after removing from the app monitor
4. Revert IAM policy changes if they were overly broad
5. Document the creation attempt and outcome

## Output Format

```yaml
root_cause: "app_monitor_creation — <specific_cause>"
evidence:
  - type: creation_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix creation parameters"
  long_term: "Deploy app monitors via IaC with validated configurations"
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
