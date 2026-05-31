---
title: "D3 — Evidence Quality"
description: "Diagnose evidence quality and completeness issues"
status: active
severity: MEDIUM
triggers: ["evidence incomplete", "evidence quality", "insufficient evidence"]
owner: devops-agent
objective: "Improve evidence quality and completeness"
context: "Evidence quality affects audit readiness. Incomplete or irrelevant evidence requires additional collection or manual supplementation."
---

## Phase 1 — Triage

MUST:
- Review evidence folders for completeness
- Check evidence relevance to control requirements
- Verify evidence covers the assessment period

SHOULD:
- Identify controls with insufficient evidence
- Check if manual evidence supplements are needed

MAY:
- Compare evidence quality across control sets

## Phase 2 — Remediate

MUST:
- Supplement insufficient evidence with manual uploads
- Ensure evidence covers the full assessment period

SHOULD:
- Add context/comments to evidence items
- Document evidence collection gaps

MAY:
- Implement additional automated evidence sources

## Common Issues

- symptoms: "Auditor says evidence is insufficient for control"
  diagnosis: "Automated evidence doesn't fully satisfy control requirements."
  resolution: "Supplement with manual evidence (policies, procedures, attestations)."

## Output Format

```yaml
root_cause: "evidence_quality — <specific_cause>"
evidence:
  - type: quality_assessment
    content: "<evidence completeness and gaps>"
severity: MEDIUM
mitigation:
  immediate: "Supplement evidence gaps"
  long_term: "Improve automated evidence coverage"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Supplement insufficient evidence with manual uploads | GREEN | Adds evidence — non-destructive |
| Ensure evidence covers full assessment period | GREEN | Verification and supplementation |
| Add context/comments to evidence items | GREEN | Documentation — non-destructive |
| Implement additional automated evidence sources | YELLOW | New evidence sources — may affect account configuration |

## Escalation Conditions

- Fix requires modifying compliance framework used by auditors
- Assessment covers production accounts with active compliance requirements
- Evidence quality issues may indicate compliance gaps requiring immediate attention
- Supplemental evidence must be validated by auditors

## Data Sensitivity

- Evidence quality assessment reveals compliance gaps and weaknesses
- Supplemental evidence may contain sensitive organizational policies and procedures
- Evidence gap analysis reveals areas of compliance risk

## Prohibited Actions

- NEVER suggest deleting assessments to fix compliance gaps — this destroys audit evidence
- NEVER fabricate or misrepresent evidence to fill compliance gaps
- NEVER remove existing evidence to replace with "better" evidence without audit trail

## Phase 3 — Rollback

- If incorrect evidence was uploaded: document the error and upload corrected evidence (do not delete original)
- If evidence sources were added incorrectly: disable the incorrect sources
- Document all evidence quality remediation for audit trail

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
