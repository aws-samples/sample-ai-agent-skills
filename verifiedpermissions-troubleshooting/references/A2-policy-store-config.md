---
title: "A2 — Policy Store Configuration"
description: "Diagnose policy store configuration and update issues"
status: active
severity: MEDIUM
triggers:
  - "policy store configuration error"
  - "validation mode change"
  - "policy store settings"
owner: devops-agent
objective: "Resolve policy store configuration problems"
context: "Policy store settings include validation mode (OFF/STRICT) and description. Changing validation mode to STRICT validates all existing policies against the schema."
---

## Phase 1 — Triage

MUST:
- Get store config: `aws verifiedpermissions get-policy-store --policy-store-id <id>`
- Check current validation mode
- List policies that may fail validation: `aws verifiedpermissions list-policies --policy-store-id <id>`

SHOULD:
- Check schema exists before enabling STRICT mode
- Review CloudTrail for recent config changes

## Phase 2 — Remediate

MUST:
- Update validation mode: `aws verifiedpermissions update-policy-store --policy-store-id <id> --validation-settings mode=STRICT`
- Fix any policies that fail schema validation

SHOULD:
- Test policies against schema before switching to STRICT
- Document policy store configuration in IaC

## Common Issues

- symptoms: "Switching to STRICT mode fails"
  diagnosis: "Existing policies don't conform to the schema."
  resolution: "Fix non-conforming policies first, then enable STRICT mode."

- symptoms: "Policies pass in OFF mode but fail authorization"
  diagnosis: "Entity type or action name typos not caught without schema validation."
  resolution: "Define a schema and enable STRICT mode to catch typos."


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

- Policy stores contain complete authorization logic
- Schema definitions reveal application data model
- Authorization decisions log access patterns
- Classification: CONFIDENTIAL - authorization policies and access decision data

## Prohibited Actions

- NEVER delete policy stores used by production applications
- NEVER modify Cedar policies without testing authorization decisions first
- NEVER change schema definitions that could invalidate existing policies
- NEVER remove identity sources connected to active authentication flows

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
