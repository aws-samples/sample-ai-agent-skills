---
title: "B1 — SSML Validation Errors"
description: "Diagnose SSML markup validation failures"
status: active
severity: MEDIUM
triggers: ["SSML error", "invalid markup", "XML parse error"]
owner: devops-agent
objective: "Identify and resolve SSML validation errors"
context: "SSML errors occur when: missing speak tags, unclosed tags, invalid attributes, or unsupported tags."
---

## Phase 1 — Triage

MUST:
- Verify SSML is wrapped in <speak></speak> tags
- Check all tags are properly closed (self-closing or paired)
- Verify XML is well-formed (escape &, <, > characters)
- Check tag names and attributes against Polly supported SSML

SHOULD:
- Validate SSML with an XML validator
- Check for unsupported SSML tags
- Verify attribute values are within allowed ranges

MAY:
- Test with minimal SSML: `<speak>Hello world</speak>`
- Add SSML tags incrementally to find the failing tag

## Phase 2 — Remediate

MUST:
- Wrap content in <speak></speak> tags
- Close all tags properly: `<break time="500ms"/>`
- Escape special XML characters

SHOULD:
- Use only Polly-supported SSML tags
- Validate SSML before API calls

## Output Format

```yaml
root_cause: "ssml_error — <specific_cause>"
evidence:
  - type: ssml_content
    content: "<failing SSML>"
severity: MEDIUM
```


## Safety Ratings
- Overall Risk: LOW
- Data Exposure Risk: LOW — SSML is markup for speech synthesis
- Service Disruption Risk: LOW — SSML errors affect individual synthesis requests
- Compliance Impact: LOW — SSML is a development concern

## Escalation Conditions
- SSML parsing fails for all requests
- SSML tags produce unexpected speech output
- SSML validation changes break existing templates
- Complex SSML causes synthesis timeouts
- SSML injection detected in user-provided content

## Data Sensitivity
- SSML templates — INTERNAL
- Text content within SSML — CONFIDENTIAL (may contain sensitive data)
- SSML configurations — INTERNAL
- Error messages — INTERNAL
- Never expose SSML content containing sensitive text

## Prohibited Actions
- Do NOT accept unvalidated SSML from user input (SSML injection risk)
- Do NOT ignore SSML validation errors
- Do NOT use deprecated SSML tags
- Do NOT bypass SSML sanitization for user-provided content
- Do NOT create SSML that produces misleading speech
- Do NOT ignore SSML compatibility warnings across voices

## Phase 3 — Rollback
- Restore previous SSML templates
- Revert SSML tag changes
- Restore SSML validation configurations
- Verify speech output after SSML changes
- Document SSML issues and tag compatibility
- Update SSML template management procedures

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
