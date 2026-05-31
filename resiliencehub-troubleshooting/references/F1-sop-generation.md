---
title: "F1 — SOP Generation"
description: "Diagnose SOP generation issues"
status: active
severity: MEDIUM
triggers: ["SOP generation failed", "no SOPs generated", "runbook error"]
owner: devops-agent
objective: "Resolve SOP generation issues"
context: "Resilience Hub generates SOPs as SSM documents for recovery procedures. Generation requires a successful assessment."
---

## Phase 1 — Triage

MUST:
- List SOP recommendations: `aws resiliencehub list-sop-recommendations --assessment-arn <arn>`
- Verify assessment completed successfully
- Check if SOPs were generated for the assessment

SHOULD:
- Verify SSM permissions for creating documents
- Check if SOP type matches the disruption scenario

MAY:
- Review generated SOP content for completeness

## Phase 2 — Remediate

MUST:
- Run successful assessment before requesting SOPs
- Ensure IAM permissions include ssm:CreateDocument

SHOULD:
- Customize generated SOPs before use
- Test SOPs in non-production environment

MAY:
- Create custom SOPs for scenarios not covered by generation

## Common Issues

- symptoms: "No SOP recommendations in assessment"
  diagnosis: "Assessment may not have identified recoverable components."
  resolution: "Verify application has components that support SOP generation."

## Output Format

```yaml
root_cause: "sop_generation — <specific_cause>"
evidence:
  - type: sop_status
    content: "<SOP generation results>"
severity: MEDIUM
mitigation:
  immediate: "Fix generation prerequisites"
  long_term: "Maintain and test SOPs regularly"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Application architecture details reveal infrastructure topology
- Assessment results may expose security weaknesses
- RTO/RPO configurations reveal business continuity priorities
- Classification: INTERNAL - contains infrastructure architecture details

## Prohibited Actions

- NEVER delete resiliency policies attached to production applications
- NEVER modify RTO/RPO targets without stakeholder approval
- NEVER dismiss assessment recommendations without documented justification
- NEVER remove application resources from resilience assessments without review

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

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
