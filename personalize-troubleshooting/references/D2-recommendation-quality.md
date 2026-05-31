---
title: "D2 — Recommendation Quality Issues"
description: "Diagnose poor recommendation quality"
status: active
severity: MEDIUM
triggers: ["recommendations irrelevant", "poor quality", "not personalized"]
owner: devops-agent
objective: "Improve recommendation quality"
context: "Recommendation quality depends on data volume, data quality, recipe selection, and model freshness. More diverse interactions produce better personalization."
---

## Phase 1 — Triage

MUST:
- Check solution version metrics: `aws personalize describe-solution-version --solution-version-arn <arn>`
- Review training metrics (coverage, precision, NDCG)
- Check data volume and quality
- Verify recipe matches use case

SHOULD:
- Check model freshness (when was it last trained)
- Review interaction data diversity
- Test with different users

MAY:
- Compare metrics across solution versions
- Analyze recommendation diversity

## Phase 2 — Remediate

MUST:
- Add more diverse interaction data
- Retrain with fresh data regularly
- Use appropriate recipe for use case

SHOULD:
- Add user and item metadata datasets
- Tune hyperparameters (exploration weight for cold start)
- Implement A/B testing

MAY:
- Build recommendation quality monitoring

## Common Issues

- symptoms: "Same recommendations for all users"
  diagnosis: "Insufficient personalization data or wrong recipe."
  resolution: "Add more per-user interactions. Check recipe selection."

- symptoms: "Recommendations don't include new items"
  diagnosis: "Model not retrained after adding new items."
  resolution: "Retrain solution version. Use exploration for cold start items."

## Output Format

```yaml
root_cause: "recommendation_quality — <specific_cause>"
evidence:
  - type: training_metrics
    content: "<solution version metrics>"
severity: MEDIUM
mitigation:
  immediate: "Retrain with more data"
  long_term: "Implement regular retraining and quality monitoring"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: HIGH — quality analysis involves user behavioral data
- Service Disruption Risk: LOW — quality issues affect relevance, not availability
- Compliance Impact: HIGH — recommendation quality affects fairness and bias compliance

## Escalation Conditions
- Recommendation quality drops significantly
- Bias detected in recommendation results
- Cold start problem affects new user experience
- Recommendation diversity too low
- A/B test shows quality regression

## Data Sensitivity
- Quality metrics — INTERNAL
- User behavioral analysis — RESTRICTED
- Bias detection results — CONFIDENTIAL
- A/B test results — INTERNAL
- Never expose individual user recommendation analysis

## Prohibited Actions
- Do NOT ignore recommendation quality degradation
- Do NOT deploy models with known bias issues
- Do NOT analyze individual user recommendations without privacy safeguards
- Do NOT share quality analysis containing user data
- Do NOT bypass quality gates for production deployment
- Do NOT ignore cold start issues for new users

## Phase 3 — Rollback
- Revert to previous model version with better quality
- Restore recommendation filters and business rules
- Re-train model with improved data
- Verify quality metrics after rollback
- Restore A/B test configurations
- Document quality issues and improvement plans

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
