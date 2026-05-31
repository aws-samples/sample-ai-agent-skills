---
title: "C2 — Voice Selection Issues"
description: "Diagnose voice selection and availability problems"
status: active
severity: LOW
triggers: ["voice not found", "wrong language", "voice quality"]
owner: devops-agent
objective: "Identify and resolve voice selection issues"
context: "Voice selection issues occur when: voice ID wrong, language mismatch, voice not in region, or engine not specified."
---

## Phase 1 — Triage

MUST:
- List available voices: `aws polly describe-voices --language-code <code>`
- Verify voice ID is case-sensitive match
- Check voice supports desired engine (standard, neural, long-form, generative)
- Verify voice is available in current region

SHOULD:
- Compare multiple voices for quality
- Check if bilingual voice is needed
- Verify gender preference availability

## Phase 2 — Remediate

MUST:
- Use exact voice ID from describe-voices output
- Specify correct engine parameter
- Use voice available in current region

SHOULD:
- Test multiple voices before selecting for production
- Document voice selection rationale

## Output Format

```yaml
root_cause: "voice_selection — <specific_cause>"
evidence:
  - type: voice_list
    content: "<available voices>"
severity: LOW
```


## Safety Ratings
- Overall Risk: LOW
- Data Exposure Risk: LOW — voice selection is a configuration concern
- Service Disruption Risk: LOW — voice changes affect speech output
- Compliance Impact: LOW — voice selection is an operational concern

## Escalation Conditions
- Selected voice not available in target region
- Voice change causes application compatibility issues
- Voice quality does not meet requirements
- Voice deprecation affects production applications
- Multiple voice changes needed simultaneously

## Data Sensitivity
- Voice configurations — INTERNAL
- Language and locale settings — INTERNAL
- Voice availability data — INTERNAL
- Application voice preferences — INTERNAL
- Never expose voice selection that reveals application strategy

## Prohibited Actions
- Do NOT use deprecated voices for new applications
- Do NOT change production voices without testing
- Do NOT ignore voice deprecation notices
- Do NOT select voices without verifying language support
- Do NOT bypass voice quality validation
- Do NOT use voices for generating misleading content

## Phase 3 — Rollback
- Restore previous voice selections
- Revert language and locale changes
- Verify speech output after voice changes
- Restore application voice configurations
- Document voice selection issues
- Update voice selection procedures

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
