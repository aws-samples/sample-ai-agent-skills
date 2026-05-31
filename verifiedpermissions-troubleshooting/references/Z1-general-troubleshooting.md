---
title: "Z1 — General Troubleshooting"
description: "Catch-all for Verified Permissions issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers:
  - "Verified Permissions error"
  - "something wrong with Verified Permissions"
  - "authorization not working"
owner: devops-agent
objective: "Systematic approach to unclassified Verified Permissions issues"
context: "Use this runbook when the issue doesn't clearly match a specific category. Follow the systematic triage to identify the problem domain, then hand off to the appropriate specific runbook."
---

## Phase 1 — Triage

MUST:
- Check policy store status: `aws verifiedpermissions list-policy-stores`
- Check schema: `aws verifiedpermissions get-schema --policy-store-id <id>`
- List policies: `aws verifiedpermissions list-policies --policy-store-id <id>`
- List identity sources: `aws verifiedpermissions list-identity-sources --policy-store-id <id>`
- Test authorization: `aws verifiedpermissions is-authorized --policy-store-id <id> --principal <json> --action <json> --resource <json>`
- Check CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=verifiedpermissions.amazonaws.com --max-results 10`

SHOULD:
- Check IAM permissions for the calling identity
- Verify region configuration
- Review recent changes in CloudTrail

## Phase 2 — Route to specific runbook

- Policy store issues → A1, A2
- Cedar policy issues → B1, B2
- Schema issues → C1, C2
- Authorization decision issues → D1, D2
- Identity source issues → E1, E2
- Template issues → F1, F2

## Common Issues

- symptoms: "AccessDeniedException on any API call"
  diagnosis: "IAM permissions missing for Verified Permissions APIs."
  resolution: "Add verifiedpermissions:* permissions to the caller's IAM policy."

- symptoms: "ResourceNotFoundException"
  diagnosis: "Policy store, policy, or identity source ID is incorrect or deleted."
  resolution: "Verify resource IDs with list commands. Check if resource was deleted."


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
