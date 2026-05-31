---
title: "H1 — Domain Recommender Issues"
description: "Diagnose domain-specific recommender problems"
status: active
severity: MEDIUM
triggers: ["recommender error", "domain recommender failed", "CreateRecommender error"]
owner: devops-agent
objective: "Identify and resolve domain recommender issues"
context: "Domain recommenders are pre-configured for specific domains (VIDEO_ON_DEMAND, ECOMMERCE). They simplify setup but have specific data requirements per domain."
---

## Phase 1 — Triage

MUST:
- List recommenders: `aws personalize list-recommenders --dataset-group-arn <arn>`
- Check recommender status: `aws personalize describe-recommender --recommender-arn <arn>`
- Verify dataset group uses domain dataset group
- Check domain-specific data requirements

SHOULD:
- Verify recipe matches domain use case
- Check dataset schema matches domain requirements
- Review recommender configuration

MAY:
- Compare with custom solution approach
- Check recommender metrics

## Phase 2 — Remediate

MUST:
- Use domain dataset group (not custom)
- Meet domain-specific data requirements
- Use domain-appropriate recipe

SHOULD:
- Follow domain-specific schema requirements
- Include required metadata fields

MAY:
- Implement domain recommender monitoring

## Common Issues

- symptoms: "CreateRecommender fails"
  diagnosis: "Dataset group not domain type or missing required datasets."
  resolution: "Use domain dataset group. Create required datasets for the domain."

- symptoms: "Recommender active but poor quality"
  diagnosis: "Insufficient domain-specific data."
  resolution: "Add more domain-relevant interaction and metadata data."

## Output Format

```yaml
root_cause: "domain_recommender — <specific_cause>"
evidence:
  - type: recommender_status
    content: "<recommender details>"
severity: MEDIUM
mitigation:
  immediate: "Fix dataset group or data requirements"
  long_term: "Implement domain recommender monitoring"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: HIGH — domain recommenders process user behavioral data
- Service Disruption Risk: MEDIUM — recommender failures affect user experience
- Compliance Impact: HIGH — recommendations must comply with domain-specific regulations

## Escalation Conditions
- Domain recommender becomes unavailable
- Recommender produces irrelevant recommendations
- Recommender latency exceeds SLA thresholds
- Recommender costs exceed budget
- Domain-specific compliance requirements not met

## Data Sensitivity
- User interaction data — RESTRICTED
- Recommendation outputs — CONFIDENTIAL
- Domain-specific configurations — INTERNAL
- Recommender metrics — INTERNAL
- Never expose individual user recommendations

## Prohibited Actions
- Do NOT deploy domain recommenders without quality validation
- Do NOT ignore recommender quality degradation
- Do NOT serve recommendations without user consent
- Do NOT expose recommender endpoints without authentication
- Do NOT bypass domain-specific compliance requirements
- Do NOT share recommender configurations externally

## Phase 3 — Rollback
- Revert to previous recommender configuration
- Restore recommendation filters
- Verify recommendation quality after rollback
- Restore monitoring configurations
- Document recommender issues and resolution steps
- Update domain recommender management procedures

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
