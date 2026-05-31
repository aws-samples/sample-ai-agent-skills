---
title: "B2 — Targeted Sentiment Issues"
description: "Diagnose targeted sentiment analysis problems"
status: active
severity: MEDIUM
triggers:
  - "targeted sentiment error"
  - "entity sentiment wrong"
  - "DetectTargetedSentiment error"
owner: devops-agent
objective: "Identify and resolve targeted sentiment issues"
context: "DetectTargetedSentiment identifies sentiment toward specific entities in text. Returns entity mentions with associated sentiment. More granular than document-level sentiment."
---

## Phase 1 — Triage

MUST:
- Test targeted sentiment: `aws comprehend detect-targeted-sentiment --text "The food was great but the service was terrible" --language-code en`
- Review entity mentions and their sentiment
- Verify language code (English only for targeted sentiment)
- Check text size limits

SHOULD:
- Verify entities are detected in text
- Check sentiment scores per entity
- Test with text containing multiple entities

MAY:
- Compare with document-level sentiment
- Use batch API for multiple documents

## Phase 2 — Remediate

MUST:
- Use English language code (en) — targeted sentiment is English-only
- Ensure text contains identifiable entities
- Handle per-entity sentiment in response

SHOULD:
- Filter by entity confidence and sentiment confidence
- Aggregate entity-level sentiment for reporting

MAY:
- Combine with entity recognition for comprehensive analysis

## Common Issues

- symptoms: "No entities returned in targeted sentiment"
  diagnosis: "Text doesn't contain recognizable entities."
  resolution: "Ensure text mentions specific entities (people, products, etc.)."

- symptoms: "Targeted sentiment not available for language"
  diagnosis: "Targeted sentiment only supports English."
  resolution: "Use document-level DetectSentiment for other languages."

## Output Format

```yaml
root_cause: "targeted_sentiment — <specific_cause>"
evidence:
  - type: targeted_result
    content: "<DetectTargetedSentiment response>"
severity: MEDIUM
mitigation:
  immediate: "Fix language or text input"
  long_term: "Implement entity-level sentiment pipeline"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | MEDIUM — Entity-level sentiment may reveal opinions about specific individuals or products |
| Blast Radius | LOW — Read-only API; no data modification |
| Rollback Complexity | LOW — Stateless API calls |
| Compliance Scope | MEDIUM — Per-entity sentiment used in profiling may trigger privacy regulations |

## Escalation Conditions

- Escalate to legal/compliance if entity-level sentiment is used to profile individuals
- Escalate to AWS Support if targeted sentiment fails consistently for valid English text
- Escalate to data governance if results are stored without proper access controls

## Data Sensitivity

- **Entity-level sentiment** links opinions to specific named entities — treat as potentially sensitive
- **Input text** may reference real people, organizations, or products
- Results should be access-controlled when they contain sentiment about identifiable individuals

## PROHIBITED Actions

- Do NOT use entity-level sentiment to build individual profiles without privacy review
- Do NOT process non-English text with targeted sentiment (English-only API) and present results as valid
- Do NOT store entity sentiment results in unprotected data stores
- Do NOT use targeted sentiment results for discriminatory decision-making

## Phase 3 — Rollback

- No rollback needed — stateless read-only API
- If incorrect results were consumed downstream, notify consumers and flag affected records
- Revert to document-level sentiment if targeted sentiment produces unreliable results for the use case

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
