---
title: "Z1 — General AppConfig Troubleshooting"
description: "Catch-all troubleshooting for AppConfig issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers:
  - "appconfig error"
  - "appconfig issue"
  - "configuration management problem"
owner: devops-agent
objective: "Diagnose and resolve general AppConfig issues"
context: "General troubleshooting for AppConfig when the specific issue category is unclear."
---

## Phase 1 — Triage

MUST:
- Identify the AppConfig component involved (application, environment, profile, deployment, extension)
- Check CloudTrail for recent AppConfig API calls: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=appconfig.amazonaws.com`
- Verify IAM permissions for the calling principal
- Check service quotas: `aws service-quotas list-service-quotas --service-code appconfig`

SHOULD:
- Review recent deployments for the affected application/environment
- Check CloudWatch metrics for AppConfig

MAY:
- Review AWS Health Dashboard for AppConfig service issues

## Phase 2 — Remediate

MUST:
- Address the specific error identified during triage
- Verify the fix resolves the issue

SHOULD:
- Document the root cause and resolution
- Add monitoring for the failure mode

MAY:
- Create automation to prevent recurrence

safety_ratings:
  - "CloudTrail lookup: GREEN"
  - "Service quota check: GREEN"
  - "IAM permission changes: YELLOW"

escalation_conditions:
  - "Issue affects multiple applications or environments"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "AWS service issue suspected"

data_sensitivity:
  - command: "cloudtrail lookup-events"
    sensitivity: HIGH
    contains: "API calls, user identities, request parameters"

prohibited_actions:
  - "NEVER modify production configurations without following the deployment process"
  - "NEVER bypass validation to force a configuration change"

## Phase 3 — Rollback

- Revert any changes made during troubleshooting
- Restore previous configuration versions if needed

## Common Issues

- symptoms: "ThrottlingException on AppConfig API calls"
  diagnosis: "API rate limit exceeded"
  resolution: "Implement exponential backoff, reduce polling frequency, or request quota increase"

- symptoms: "AccessDeniedException on AppConfig operations"
  diagnosis: "IAM policy missing required AppConfig permissions"
  resolution: "Add the specific appconfig: actions needed to the IAM policy"

## Output Format

```yaml
root_cause: "<description>"
evidence:
  - type: general
    content: "<diagnostic findings>"
severity: MEDIUM
mitigation:
  immediate: "<fix>"
  long_term: "<prevention>"
```
