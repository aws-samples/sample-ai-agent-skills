---
title: "D1 — Health API"
description: "Diagnose Health API access and usage issues"
status: active
severity: MEDIUM
triggers:
  - "Health API error"
  - "cannot access Health API"
  - "describe-events failed"
  - "Health API permission"
owner: devops-agent
objective: "Resolve Health API access and usage issues"
context: "The Health API is part of the AWS Health service. It requires Business/Enterprise Support, us-east-1 endpoint, and IAM permissions for health:Describe* actions. The API returns events for all regions from the us-east-1 endpoint."
---

## Phase 1 — Triage

MUST:
- Verify API endpoint is us-east-1: ensure --region us-east-1 is specified
- Check IAM permissions: `aws iam simulate-principal-policy --policy-source-arn <arn> --action-names health:DescribeEvents`
- Verify support plan allows API access (Business/Enterprise)
- Test basic API call: `aws health describe-events --region us-east-1`

SHOULD:
- Check for SCP blocking health:* actions
- Verify no permission boundary restricting Health API
- Review CloudTrail for authorization failures

MAY:
- Check if API rate limits are being hit
- Verify SDK/CLI version supports the Health API

## Phase 2 — Remediate

MUST:
- Add IAM permissions for health:Describe* actions
- Use --region us-east-1 for all API calls
- Ensure Business or Enterprise Support plan

SHOULD:
- Create dedicated IAM policy for Health API access
- Remove SCP restrictions if blocking legitimate access
- Implement proper error handling in API automation

MAY:
- Set up IAM role for cross-account Health API access
- Implement API response caching for frequently queried events

## Common Issues

- symptoms: "Could not connect to the endpoint URL"
  diagnosis: "API call not targeting us-east-1."
  resolution: "Add --region us-east-1 to all Health API calls."

- symptoms: "SubscriptionRequiredException"
  diagnosis: "Account has Basic or Developer Support plan."
  resolution: "Upgrade to Business or Enterprise Support for API access."

- symptoms: "AccessDeniedException"
  diagnosis: "IAM policy missing health:Describe* permissions."
  resolution: "Add health:Describe* to IAM policy."

## Output Format

```yaml
root_cause: "health_api — <specific_cause>"
evidence:
  - type: api_error
    content: "<specific API error message>"
  - type: permissions
    content: "<IAM policy and support plan>"
severity: MEDIUM
mitigation:
  immediate: "Fix region, permissions, or support plan"
  long_term: "Implement dedicated IAM policy and error handling"
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
- Never dismiss or archive health events without documenting their resolution
- Never disable EventBridge rules for health notifications in production accounts
- Never ignore scheduled maintenance events without planning for impact
- Never delete organizational health view configurations without stakeholder approval
- Never modify health event filters that may suppress critical notifications

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Re-enable EventBridge notification rules if health alerts stop flowing
- Restore previous event filter configurations if critical events are missed
- Revert organizational view delegation changes if visibility is lost

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
