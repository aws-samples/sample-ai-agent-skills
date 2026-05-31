---
title: "A1 — Policy Store Creation"
description: "Diagnose policy store creation and setup failures"
status: active
severity: HIGH
triggers:
  - "policy store creation failed"
  - "cannot create policy store"
  - "ValidationException on CreatePolicyStore"
owner: devops-agent
objective: "Identify and resolve policy store creation issues"
context: "A policy store is the container for Cedar policies, schema, identity sources, and policy templates. Each store is regional. Validation mode can be OFF or STRICT."
---

## Phase 1 — Triage

MUST:
- List existing policy stores: `aws verifiedpermissions list-policy-stores`
- Check IAM permissions for the caller: `aws sts get-caller-identity`
- Verify region: `aws configure get region`
- Check CloudTrail for creation errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreatePolicyStore --max-results 5`

SHOULD:
- Verify service quotas for policy stores per region
- Check if validation mode setting is correct (OFF or STRICT)

## Phase 2 — Remediate

MUST:
- Create policy store: `aws verifiedpermissions create-policy-store --validation-settings mode=STRICT`
- Verify creation: `aws verifiedpermissions get-policy-store --policy-store-id <id>`

SHOULD:
- Use STRICT validation mode for production stores
- Add a schema immediately after creation

## Common Issues

- symptoms: "AccessDeniedException on CreatePolicyStore"
  diagnosis: "Caller lacks verifiedpermissions:CreatePolicyStore permission."
  resolution: "Add verifiedpermissions:CreatePolicyStore to the IAM policy."

- symptoms: "ServiceQuotaExceededException"
  diagnosis: "Maximum policy stores per region reached."
  resolution: "Delete unused policy stores or request a quota increase."


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
