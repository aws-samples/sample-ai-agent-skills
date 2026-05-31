---
title: "B1 — Sentiment Analysis Issues"
description: "Diagnose sentiment analysis problems"
status: active
severity: MEDIUM
triggers:
  - "sentiment wrong"
  - "DetectSentiment error"
  - "sentiment inaccurate"
owner: devops-agent
objective: "Identify and resolve sentiment analysis issues"
context: "DetectSentiment returns POSITIVE, NEGATIVE, NEUTRAL, MIXED with probability scores. Requires language code. Works best with opinion-bearing text."
---

## Phase 1 — Triage

MUST:
- Test sentiment: `aws comprehend detect-sentiment --text "This product is amazing" --language-code en`
- Verify language code matches text language
- Review all four sentiment scores
- Check text size limits

SHOULD:
- Test with clearly positive and negative text
- Check if text is opinion-bearing (factual text returns NEUTRAL)
- Verify text encoding

MAY:
- Use targeted sentiment for entity-level analysis
- Compare batch vs real-time results

## Phase 2 — Remediate

MUST:
- Use correct language code
- Understand four-category output (not binary)
- Handle MIXED sentiment appropriately

SHOULD:
- Use dominant sentiment score for confidence assessment
- Implement threshold for sentiment classification

MAY:
- Train custom classifier for domain-specific sentiment

## Common Issues

- symptoms: "Sentiment always returns NEUTRAL"
  diagnosis: "Text is factual/informational, not opinion-bearing."
  resolution: "Sentiment analysis works best on opinion text. Use custom classifier for domain-specific needs."

- symptoms: "Sentiment seems wrong for domain-specific text"
  diagnosis: "Built-in model not trained on domain-specific language."
  resolution: "Train custom classifier with domain-specific labeled data."

## Output Format

```yaml
root_cause: "sentiment_analysis — <specific_cause>"
evidence:
  - type: sentiment_result
    content: "<DetectSentiment response>"
severity: MEDIUM
mitigation:
  immediate: "Fix language code or text input"
  long_term: "Train custom classifier for domain-specific sentiment"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | MEDIUM — Input text may contain opinions about individuals or confidential topics |
| Blast Radius | LOW — Read-only API; no data modification |
| Rollback Complexity | LOW — Stateless API calls |
| Compliance Scope | MEDIUM — Sentiment results used in decision-making may have regulatory implications |

## Escalation Conditions

- Escalate to legal/compliance if sentiment analysis results are used for automated decisions about individuals (e.g., employee reviews, customer scoring)
- Escalate to AWS Support if sentiment consistently returns incorrect results for clearly positive/negative text
- Escalate to data governance if sentiment analysis is applied to data outside its approved use case

## Data Sensitivity

- **Input text** may contain personal opinions, employee feedback, or customer complaints — handle per data classification
- **Sentiment scores** can reveal attitudes about individuals or organizations — restrict access appropriately
- **Batch job outputs** in S3 must be encrypted and access-controlled
- Do not log raw input text in troubleshooting artifacts

## PROHIBITED Actions

- Do NOT use sentiment analysis results as the sole basis for automated decisions affecting individuals without human review
- Do NOT process regulated communications (e.g., healthcare patient feedback) without proper data handling agreements
- Do NOT share sentiment analysis results outside authorized business functions
- Do NOT disable encryption on S3 buckets storing batch sentiment results

## Phase 3 — Rollback

- No rollback needed — DetectSentiment is a stateless read-only API
- If batch sentiment jobs produced incorrect results, delete the output from S3 and rerun with corrected parameters
- If downstream systems consumed incorrect sentiment data, notify consumers and provide corrected results

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
