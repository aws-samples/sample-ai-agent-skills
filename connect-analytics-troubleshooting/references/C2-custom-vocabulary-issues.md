---
title: "C2 — Custom Vocabulary Issues"
description: "Diagnose Contact Lens custom vocabulary configuration issues"
status: active
severity: MEDIUM
triggers: ["custom vocabulary error", "vocabulary not applied", "wrong transcription", "term not recognized"]
owner: devops-agent
objective: "Resolve custom vocabulary issues to improve transcription accuracy"
context: "Custom vocabulary improves transcription accuracy for domain-specific terms, product names, and jargon. Vocabulary files have specific format requirements and size limits. Changes apply to future contacts only."
---

## Phase 1 — Triage

MUST:
- Check vocabulary status: `aws connect list-default-vocabularies --instance-id <instance-id> --language-code en-US`
- Get vocabulary details: `aws connect describe-vocabulary --instance-id <instance-id> --vocabulary-id <vocabulary-id>`
- Verify the vocabulary state is ACTIVE (not CREATION_IN_PROGRESS or CREATION_FAILED)

SHOULD:
- Check the vocabulary file format (Phrase, IPA, SoundsLike, DisplayAs columns)
- Verify the vocabulary language matches the contact flow language
- Test transcription accuracy with and without the vocabulary

MAY:
- Review the vocabulary file for formatting errors
- Check CloudTrail for vocabulary creation events

## Phase 2 — Remediate

MUST:
- Fix vocabulary file format issues (correct column headers, valid entries)
- Create or update vocabulary: `aws connect create-vocabulary --instance-id <instance-id> --vocabulary-name <name> --language-code en-US --content <vocabulary-content>`
- Wait for vocabulary state to become ACTIVE before testing

SHOULD:
- Set the vocabulary as default for the language: `aws connect associate-default-vocabulary --instance-id <instance-id> --language-code en-US --vocabulary-id <vocabulary-id>`
- Test with a new contact after vocabulary activation
- Keep vocabulary entries focused on frequently misrecognized terms

MAY:
- Create separate vocabularies for different business domains
- Implement vocabulary update procedures for new product launches
- Monitor transcription accuracy metrics over time

## Safety Ratings

```
safety_ratings:
  - "list-default-vocabularies: GREEN — read-only vocabulary listing"
  - "describe-vocabulary: GREEN — read-only vocabulary details"
  - "create-vocabulary: YELLOW — creates vocabulary resource"
  - "associate-default-vocabulary: YELLOW — changes transcription behavior for all contacts"
  - "delete-vocabulary: RED — removes vocabulary permanently"
```

## Escalation Conditions

- Vocabulary creation fails repeatedly with no clear error
- Vocabulary is ACTIVE but transcription accuracy does not improve
- Vocabulary size limit prevents adding all required terms
- Vocabulary causes transcription degradation for common words

## Data Sensitivity

- `aws connect list-default-vocabularies`: LOW — returns vocabulary metadata
- `aws connect describe-vocabulary`: LOW — returns vocabulary configuration
- Vocabulary content: LOW — contains domain-specific terms (not customer data)
- Transcription output: HIGH — contains customer conversation content

## Prohibited Actions

- NEVER delete the active default vocabulary without a replacement
- NEVER add customer PII to vocabulary entries
- NEVER assume vocabulary changes apply retroactively to past contacts
- NEVER create vocabularies with thousands of entries without testing impact

## Phase 3 — Rollback

1. Disassociate the vocabulary if it worsened transcription: `aws connect disassociate-default-vocabulary --instance-id <instance-id> --language-code en-US`
2. Associate the previous vocabulary as default
3. Delete the problematic vocabulary: `aws connect delete-vocabulary --instance-id <instance-id> --vocabulary-id <vocabulary-id>`
4. Verify transcription accuracy returns to baseline for new contacts
5. Document the vocabulary change and rollback reason

## Common Issues

- symptoms: "Vocabulary shows CREATION_FAILED status"
  diagnosis: "Vocabulary file has formatting errors — invalid entries or unsupported characters."
  resolution: "Fix the vocabulary file format. Ensure columns are: Phrase, IPA, SoundsLike, DisplayAs."

## Output Format

```yaml
root_cause: "custom_vocabulary — <specific_cause>"
evidence:
  - type: vocabulary_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix vocabulary configuration"
  long_term: "Implement vocabulary management and testing procedures"
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
