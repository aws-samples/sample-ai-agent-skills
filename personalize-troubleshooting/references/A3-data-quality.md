---
title: "A3 — Data Quality Issues"
description: "Diagnose data quality problems affecting recommendations"
status: active
severity: MEDIUM
triggers: ["poor data quality", "insufficient data", "data requirements"]
owner: devops-agent
objective: "Identify and resolve data quality issues"
context: "Personalize requires minimum 1000 interactions, 25 unique users, 2 interactions per user. Data quality directly affects recommendation quality."
---

## Phase 1 — Triage

MUST:
- Check dataset metrics: `aws personalize describe-dataset --dataset-arn <arn>`
- Verify minimum data requirements met
- Check data recency (stale data affects quality)
- Review interaction distribution across users/items

SHOULD:
- Check for data skew (few users with many interactions)
- Verify timestamp format and range
- Check for duplicate records

MAY:
- Analyze user/item coverage
- Check event type distribution

## Phase 2 — Remediate

MUST:
- Meet minimum requirements (1000 interactions, 25 users, 2 per user)
- Use recent interaction data
- Include diverse user-item interactions

SHOULD:
- Balance interaction distribution
- Include multiple event types
- Add user and item metadata

MAY:
- Implement data quality monitoring

## Common Issues

- symptoms: "Training fails with insufficient data"
  diagnosis: "Below minimum data requirements."
  resolution: "Add more interaction data. Meet minimums before training."

- symptoms: "Recommendations are repetitive or irrelevant"
  diagnosis: "Data too sparse or skewed."
  resolution: "Add more diverse interactions. Include metadata datasets."

## Output Format

```yaml
root_cause: "data_quality — <specific_cause>"
evidence:
  - type: dataset_metrics
    content: "<dataset statistics>"
severity: MEDIUM
mitigation:
  immediate: "Add more quality data"
  long_term: "Implement data quality monitoring pipeline"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: HIGH — data quality analysis involves inspecting user data
- Service Disruption Risk: LOW — data quality issues affect model quality, not availability
- Compliance Impact: HIGH — data quality must comply with privacy and consent requirements

## Escalation Conditions
- Data quality issues cause model training failures
- PII detected in datasets that should be anonymized
- Data bias detected affecting recommendation fairness
- Data freshness issues affect recommendation relevance
- Data volume insufficient for meaningful recommendations

## Data Sensitivity
- User behavioral data — RESTRICTED
- Data quality metrics — INTERNAL
- Data distribution analysis — INTERNAL
- Bias detection results — CONFIDENTIAL
- Never expose individual user data during quality analysis

## Prohibited Actions
- Do NOT analyze individual user records without privacy safeguards
- Do NOT ignore data quality warnings before training
- Do NOT use biased datasets without mitigation
- Do NOT retain data quality analysis containing PII
- Do NOT bypass data governance review for quality issues
- Do NOT share data quality reports containing user data externally

## Phase 3 — Rollback
- Restore previous dataset versions with better quality
- Remove datasets with identified quality issues
- Revert data processing pipeline changes
- Re-run data quality checks after corrections
- Verify model performance after data quality improvements
- Document data quality issues and remediation steps

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
