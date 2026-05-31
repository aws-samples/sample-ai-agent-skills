---
title: "B2 — SSML Tag Issues"
description: "Diagnose specific SSML tag behavior problems"
status: active
severity: LOW
triggers: ["prosody not working", "break ignored", "phoneme wrong"]
owner: devops-agent
objective: "Identify and resolve SSML tag behavior issues"
context: "Tag issues occur when: attribute values out of range, tag not supported by voice, or tag combination conflicts."
---

## Phase 1 — Triage

MUST:
- Verify tag is supported by Polly (break, emphasis, lang, mark, p, phoneme, prosody, s, say-as, sub, w)
- Check attribute values are within allowed ranges
- Verify tag is compatible with the selected engine (standard vs neural)

SHOULD:
- Test each tag individually to isolate issues
- Check prosody rate/pitch/volume ranges
- Verify phoneme alphabet matches voice language

## Phase 2 — Remediate

MUST:
- Use only Polly-supported SSML tags
- Fix attribute values to within allowed ranges
- Match phoneme alphabet to voice language

SHOULD:
- Test SSML incrementally
- Document working SSML patterns

## Output Format

```yaml
root_cause: "ssml_tag — <specific_cause>"
evidence:
  - type: ssml_tag
    content: "<problematic tag>"
severity: LOW
```


## Safety Ratings
- Overall Risk: LOW
- Data Exposure Risk: LOW — SSML tags control speech output
- Service Disruption Risk: LOW — tag issues affect speech quality
- Compliance Impact: LOW — SSML tags are a development concern

## Escalation Conditions
- SSML tags not supported by selected voice
- Tag combinations cause synthesis failures
- Prosody tags produce unnatural speech
- Break tags cause excessive pauses
- Phoneme tags produce incorrect pronunciation

## Data Sensitivity
- SSML tag configurations — INTERNAL
- Speech output samples — INTERNAL
- Voice-specific tag support — INTERNAL
- Tag compatibility matrices — INTERNAL
- Never expose SSML content containing sensitive text

## Prohibited Actions
- Do NOT use unsupported SSML tags for selected voice engine
- Do NOT ignore tag compatibility warnings
- Do NOT create SSML that produces offensive or misleading speech
- Do NOT bypass tag validation
- Do NOT use excessive prosody modifications that degrade quality
- Do NOT deploy SSML changes without testing across target voices

## Phase 3 — Rollback
- Restore previous SSML tag configurations
- Revert prosody changes
- Restore phoneme configurations
- Verify speech quality after tag changes
- Document tag compatibility issues
- Update SSML tag usage guidelines

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
