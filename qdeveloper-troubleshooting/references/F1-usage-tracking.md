---
title: "F1 — Usage Tracking Issues"
description: "Diagnose usage monitoring and limit tracking"
status: active
severity: LOW
triggers:
  - "usage limit reached"
  - "how many suggestions left"
  - "usage tracking"
owner: devops-agent
objective: "Resolve usage tracking issues"
context: "Free tier has monthly limits. Pro tier has higher limits."
---

## Phase 1 — Triage

MUST:
- Check current tier (Free/Pro)
- Verify usage against limits
- Check billing period reset date

SHOULD:
- Review usage patterns
- Check for unexpected usage spikes

## Phase 2 — Remediate

MUST:
- Wait for limit reset or upgrade tier
- Verify tier assignment

SHOULD:
- Monitor usage proactively
- Plan tier based on usage patterns

## Common Issues

- symptoms: "Suggestions stopped working mid-month"
  diagnosis: "Free tier monthly limit exceeded."
  resolution: "Wait for next month or upgrade to Pro tier."

## Output Format

```yaml
root_cause: "usage_tracking — <specific_cause>"
evidence:
  - type: usage_data
    content: "<usage and limit details>"
severity: LOW
mitigation:
  immediate: "Upgrade tier if needed"
  long_term: "Monitor usage patterns"
```


## Safety Ratings
- Overall Risk: LOW
- Data Exposure Risk: MEDIUM — usage tracking captures developer activity patterns
- Service Disruption Risk: LOW — tracking issues do not affect functionality
- Compliance Impact: MEDIUM — usage tracking must comply with privacy policies

## Escalation Conditions
- Usage tracking data not being collected
- Tracking data contains PII
- Tracking data access unauthorized
- Usage metrics inconsistent
- Tracking causes performance issues

## Data Sensitivity
- Usage metrics — INTERNAL
- Developer activity patterns — CONFIDENTIAL
- Feature usage data — INTERNAL
- Tracking configurations — INTERNAL
- Never expose individual developer usage patterns

## Prohibited Actions
- Do NOT track individual developer activity without consent
- Do NOT share usage data with unauthorized parties
- Do NOT retain usage data beyond approved periods
- Do NOT use tracking data for performance evaluation without disclosure
- Do NOT ignore tracking data privacy requirements
- Do NOT expose tracking data externally

## Phase 3 — Rollback
- Restore previous tracking configurations
- Revert tracking scope changes
- Verify tracking compliance after changes
- Restore data retention settings
- Document tracking issues
- Update tracking privacy procedures

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
