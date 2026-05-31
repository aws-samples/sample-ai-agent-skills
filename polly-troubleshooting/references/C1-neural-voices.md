---
title: "C1 — Neural Voice Issues"
description: "Diagnose neural voice availability and quality issues"
status: active
severity: MEDIUM
triggers: ["neural voice error", "voice not available", "engine not supported"]
owner: devops-agent
objective: "Identify and resolve neural voice issues"
context: "Neural voice issues occur when: language not supported, voice ID wrong, engine parameter missing, or region doesn't have neural."
---

## Phase 1 — Triage

MUST:
- Check neural voice availability: `aws polly describe-voices --engine neural --language-code <code>`
- Verify voice ID supports neural engine
- Check region supports neural voices
- Verify engine parameter is set to "neural" in API call

SHOULD:
- Compare standard vs neural output for quality
- Check if generative engine is available for the voice
- Verify sample rate is appropriate for neural (24000 Hz default)

MAY:
- Test with different neural voices for the same language
- Check pricing differences between engines

## Phase 2 — Remediate

MUST:
- Use a voice that supports neural engine
- Set engine parameter to "neural" explicitly
- Fall back to standard engine if neural unavailable

SHOULD:
- Use 24000 Hz sample rate for best neural quality
- Test neural output quality before production use

## Output Format

```yaml
root_cause: "neural_voice — <specific_cause>"
evidence:
  - type: voice_list
    content: "<available neural voices>"
severity: MEDIUM
```


## Safety Ratings
- Overall Risk: LOW
- Data Exposure Risk: LOW — voice selection is a configuration concern
- Service Disruption Risk: MEDIUM — neural voice issues affect speech quality
- Compliance Impact: LOW — voice selection is an operational concern

## Escalation Conditions
- Neural voice synthesis fails consistently
- Neural voice quality degrades
- Neural voice not available in target region
- Neural voice latency exceeds thresholds
- Neural voice produces unexpected output

## Data Sensitivity
- Voice configurations — INTERNAL
- Synthesis input text — CONFIDENTIAL
- Audio output — INTERNAL
- Voice availability data — INTERNAL
- Never expose synthesis input containing sensitive data

## Prohibited Actions
- Do NOT use neural voices for generating deepfake or misleading content
- Do NOT ignore neural voice availability limitations
- Do NOT bypass neural voice rate limits
- Do NOT use neural voices without understanding cost implications
- Do NOT deploy neural voice changes without quality testing
- Do NOT ignore neural voice quality degradation

## Phase 3 — Rollback
- Revert to standard voices if neural voices are unavailable
- Restore previous voice configurations
- Verify speech quality after voice changes
- Restore rate limit configurations
- Document neural voice issues and availability
- Update voice selection guidelines

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
