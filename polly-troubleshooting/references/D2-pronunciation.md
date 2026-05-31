---
title: "D2 — Pronunciation Issues"
description: "Diagnose word pronunciation and phoneme problems"
status: active
severity: LOW
triggers: ["wrong pronunciation", "word mispronounced", "acronym wrong"]
owner: devops-agent
objective: "Identify and resolve pronunciation issues"
context: "Pronunciation issues occur when: default pronunciation wrong, lexicon not applied, SSML phoneme incorrect, or say-as type wrong."
---

## Phase 1 — Triage

MUST:
- Test pronunciation with plain text first
- Check if lexicon is applied in synthesis request
- Verify SSML <phoneme> tag if used
- Check <say-as> interpret-as type for numbers, dates, etc.

SHOULD:
- Try SSML <sub> tag for simple substitutions
- Verify IPA/x-sampa phoneme strings
- Check if word is in applied lexicon

## Phase 2 — Remediate

MUST:
- Create lexicon entry for mispronounced words
- Use SSML <phoneme> for inline pronunciation override
- Use <say-as> for numbers, dates, addresses

SHOULD:
- Test pronunciation changes before production
- Maintain lexicon for domain-specific terms

## Output Format

```yaml
root_cause: "pronunciation — <specific_cause>"
evidence:
  - type: pronunciation_test
    content: "<word and expected vs actual>"
severity: LOW
```


## Safety Ratings
- Overall Risk: LOW
- Data Exposure Risk: LOW — pronunciation is a speech quality concern
- Service Disruption Risk: LOW — pronunciation issues affect speech quality
- Compliance Impact: LOW — pronunciation is an operational concern

## Escalation Conditions
- Pronunciation errors in critical content (brand names, medical terms)
- Pronunciation changes after service updates
- Phoneme-based pronunciation not working correctly
- Multiple pronunciation issues across languages
- Pronunciation inconsistencies between voices

## Data Sensitivity
- Pronunciation rules — INTERNAL
- Phoneme configurations — INTERNAL
- Custom vocabulary — INTERNAL
- Brand name pronunciations — INTERNAL
- Never expose pronunciation rules that reveal business terminology

## Prohibited Actions
- Do NOT deploy pronunciation changes without testing
- Do NOT ignore pronunciation errors in critical content
- Do NOT bypass pronunciation validation
- Do NOT create pronunciation rules that produce offensive output
- Do NOT modify production pronunciation rules without approval
- Do NOT ignore pronunciation inconsistencies across voices

## Phase 3 — Rollback
- Restore previous pronunciation configurations
- Revert lexicon changes affecting pronunciation
- Restore phoneme configurations
- Verify pronunciation after changes
- Document pronunciation issues and corrections
- Update pronunciation management procedures

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
