---
title: "D1 — Real-Time Recommendation Issues"
description: "Diagnose real-time recommendation failures"
status: active
severity: HIGH
triggers: ["GetRecommendations error", "no recommendations returned", "recommendations empty"]
owner: devops-agent
objective: "Identify and resolve real-time recommendation issues"
context: "GetRecommendations returns personalized items for a user. Requires active campaign. User must exist in dataset or have real-time events."
---

## Phase 1 — Triage

MUST:
- Verify campaign is ACTIVE: `aws personalize describe-campaign --campaign-arn <arn>`
- Test recommendations: `aws personalize-runtime get-recommendations --campaign-arn <arn> --user-id <user-id>`
- Verify user exists in interactions dataset
- Check for filter blocking all results

SHOULD:
- Test with known user from training data
- Check numResults parameter
- Verify filter expression if applied

MAY:
- Monitor campaign latency and errors
- Check for throttling

## Phase 2 — Remediate

MUST:
- Ensure campaign is ACTIVE
- Use valid user ID from dataset
- Remove or fix overly restrictive filters

SHOULD:
- Handle cold start users (new users without history)
- Set appropriate numResults

MAY:
- Implement recommendation caching

## Common Issues

- symptoms: "GetRecommendations returns empty list"
  diagnosis: "User not in dataset, filter too restrictive, or campaign not active."
  resolution: "Verify user exists. Check filter. Confirm campaign ACTIVE."

- symptoms: "ResourceNotFoundException"
  diagnosis: "Campaign ARN incorrect or campaign deleted."
  resolution: "Verify campaign ARN. Check campaign exists."

## Output Format

```yaml
root_cause: "realtime_recommendations — <specific_cause>"
evidence:
  - type: recommendation_result
    content: "<GetRecommendations response>"
severity: HIGH
mitigation:
  immediate: "Fix campaign, user, or filter"
  long_term: "Implement recommendation monitoring"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: HIGH — recommendations are based on user behavioral data
- Service Disruption Risk: HIGH — recommendation failures affect user experience
- Compliance Impact: HIGH — recommendations must comply with privacy and consent policies

## Escalation Conditions
- Real-time recommendations completely unavailable
- Recommendation latency exceeds SLA thresholds
- Recommendations contain inappropriate or biased content
- Recommendation API returns errors consistently
- Throughput throttling affects recommendation delivery

## Data Sensitivity
- User IDs and interaction history — RESTRICTED
- Recommendation results — CONFIDENTIAL
- API request/response data — CONFIDENTIAL
- User context data — RESTRICTED
- Never log individual user recommendation results

## Prohibited Actions
- Do NOT serve recommendations without user consent verification
- Do NOT log individual user recommendation results
- Do NOT expose recommendation API without authentication
- Do NOT ignore recommendation quality degradation
- Do NOT serve recommendations from untested models
- Do NOT bypass recommendation filtering rules

## Phase 3 — Rollback
- Revert to previous campaign version for recommendations
- Restore recommendation filters
- Increase throughput if throttling caused issues
- Verify recommendation quality after rollback
- Restore fallback recommendation logic
- Document recommendation issues and resolution steps

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
