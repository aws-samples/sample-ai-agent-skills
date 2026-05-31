---
title: "D1 — Lexicon Errors"
description: "Diagnose lexicon creation and application failures"
status: active
severity: MEDIUM
triggers: ["lexicon error", "PLS invalid", "pronunciation wrong"]
owner: devops-agent
objective: "Identify and resolve lexicon errors"
context: "Lexicon errors occur when: PLS XML invalid, phoneme format wrong, lexicon too large, or lexicon limit exceeded."
---

## Phase 1 — Triage

MUST:
- Check lexicon: `aws polly get-lexicon --name <name>`
- Verify PLS XML format with proper namespace
- Check phoneme alphabet (ipa or x-sampa)
- Verify lexicon count (max 5 per synthesis request)

SHOULD:
- Validate PLS XML against schema
- Check grapheme entries for duplicates
- Verify phoneme strings are valid IPA/x-sampa

MAY:
- Test lexicon with simple synthesis
- List all lexicons: `aws polly list-lexicons`

## Phase 2 — Remediate

MUST:
- Fix PLS XML format with proper namespace declaration
- Use valid IPA or x-sampa phoneme strings
- Reduce lexicon count to 5 or fewer per request

SHOULD:
- Test pronunciation with individual entries
- Document lexicon entries and their purpose

## Output Format

```yaml
root_cause: "lexicon_error — <specific_cause>"
evidence:
  - type: lexicon_content
    content: "<PLS XML>"
severity: MEDIUM
```


## Safety Ratings
- Overall Risk: LOW
- Data Exposure Risk: LOW — lexicons contain pronunciation rules
- Service Disruption Risk: LOW — lexicon errors affect pronunciation quality
- Compliance Impact: LOW — lexicon management is a development concern

## Escalation Conditions
- Lexicon upload fails with validation errors
- Lexicon causes incorrect pronunciation
- Lexicon size exceeds limits
- Multiple lexicons conflict with each other
- Lexicon changes affect production speech output

## Data Sensitivity
- Lexicon content — INTERNAL
- Pronunciation rules — INTERNAL
- Custom vocabulary — INTERNAL
- Lexicon configurations — INTERNAL
- Never expose lexicon content that reveals business terminology

## Prohibited Actions
- Do NOT deploy untested lexicons to production
- Do NOT exceed lexicon size limits
- Do NOT create conflicting lexicon entries
- Do NOT delete lexicons used by production applications
- Do NOT bypass lexicon validation
- Do NOT share lexicon content externally

## Phase 3 — Rollback
- Restore previous lexicon versions
- Remove incorrectly uploaded lexicons
- Revert lexicon changes that caused pronunciation issues
- Verify pronunciation after lexicon changes
- Document lexicon issues and pronunciation rules
- Update lexicon management procedures

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
