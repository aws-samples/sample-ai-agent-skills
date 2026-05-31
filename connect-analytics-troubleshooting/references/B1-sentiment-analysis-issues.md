---
title: "B1 — Sentiment Analysis Issues"
description: "Diagnose Contact Lens sentiment analysis problems"
status: active
severity: MEDIUM
triggers: ["sentiment wrong", "sentiment not showing", "sentiment score incorrect", "no sentiment data"]
owner: devops-agent
objective: "Resolve sentiment analysis issues in Contact Lens"
context: "Sentiment scores range from -5 (most negative) to +5 (most positive). Scores are per-turn and overall. Sentiment depends on transcription accuracy and language support."
---

## Phase 1 — Triage

MUST:
- Verify Contact Lens is enabled and generating transcripts (sentiment requires transcription)
- Check contact analytics: `aws connect describe-contact --instance-id <instance-id> --contact-id <contact-id>`
- Verify sentiment data in post-contact analytics (check S3 output or Connect console)

SHOULD:
- Check if transcription is accurate (sentiment depends on transcript quality)
- Verify the language configured matches the actual call language
- Compare sentiment scores against manual assessment of the conversation

MAY:
- Check multiple contacts to determine if the issue is systemic or contact-specific
- Review custom vocabulary configuration that might affect transcription accuracy

## Phase 2 — Remediate

MUST:
- Ensure Contact Lens is generating transcripts (sentiment requires transcription)
- Verify the correct language is configured in the contact flow analytics block
- Confirm the contact had sufficient audio quality for accurate transcription

SHOULD:
- Add custom vocabulary for domain-specific terms that affect sentiment interpretation
- Review and calibrate sentiment expectations (scores are -5 to +5, not percentages)

MAY:
- Set up category rules based on sentiment thresholds for automated alerting
- Configure supervisor dashboards to display sentiment trends

## Safety Ratings

```
safety_ratings:
  - "describe-contact: GREEN — read-only contact details"
  - "S3 analytics output review: GREEN — read-only data access"
  - "custom vocabulary update: YELLOW — affects future transcription accuracy"
  - "category rule creation: YELLOW — creates automated evaluation rules"
```

## Escalation Conditions

- Sentiment scores consistently incorrect across multiple contacts
- Sentiment data missing for contacts that have transcripts
- Sentiment analysis not available for a supported language
- Business decisions depend on sentiment accuracy and scores are unreliable

## Data Sensitivity

- `aws connect describe-contact`: MEDIUM — returns contact metadata
- Sentiment scores: MEDIUM — reveal customer emotional state during interactions
- Transcripts: HIGH — contain full conversation content
- S3 analytics output: HIGH — contains complete contact analytics

## Prohibited Actions

- NEVER report sentiment scores as percentages — they are on a -5 to +5 scale
- NEVER claim sentiment analysis is 100% accurate — it depends on transcription quality
- NEVER use sentiment scores as the sole basis for agent performance evaluation
- NEVER access contact transcripts without authorization

## Phase 3 — Rollback

1. Revert custom vocabulary changes if they worsened transcription accuracy
2. Revert contact flow language settings if they were changed incorrectly
3. Remove category rules based on sentiment if thresholds were misconfigured
4. Document the sentiment analysis issue and calibration findings
5. Verify sentiment is being generated for new contacts after changes

## Common Issues

- symptoms: "Sentiment scores seem random and don't match the conversation tone"
  diagnosis: "Language mismatch — contact flow configured for English but calls are in Spanish."
  resolution: "Configure the correct language in the contact flow analytics block."

## Output Format

```yaml
root_cause: "sentiment_analysis — <specific_cause>"
evidence:
  - type: sentiment_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix sentiment analysis configuration"
  long_term: "Implement sentiment calibration and monitoring"
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
  - "NEVER suggest disabling call recording without compliance review"
  - "NEVER suggest removing authentication"
  - "NEVER expose customer PII in logs"
