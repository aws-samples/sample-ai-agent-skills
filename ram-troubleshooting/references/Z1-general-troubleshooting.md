---
title: "Z1 — General RAM Troubleshooting"
description: "Catch-all systematic investigation for RAM issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers:
  - "RAM issue"
  - "resource sharing error"
  - "RAM problem"
  - "sharing not working"
  - "general troubleshooting"
owner: devops-agent
objective: "Systematically investigate RAM issues using a structured diagnostic approach"
context: "This runbook provides a general-purpose investigation framework for RAM issues that don't clearly match a specific runbook. It covers resource share status, invitation checks, permission verification, and systematic elimination of common causes."
---

## Phase 1 — Triage

MUST:
- List all resource shares: `aws ram get-resource-shares --resource-owner SELF`
- List shares from others: `aws ram get-resource-shares --resource-owner OTHER-ACCOUNTS`
- Check pending invitations: `aws ram get-resource-share-invitations`
- Verify Organizations status: `aws organizations describe-organization`

SHOULD:
- Check CloudTrail for recent RAM events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=ram.amazonaws.com --max-results 20`
- List supported resource types: `aws ram list-resource-types`
- Verify RAM service-linked role
- Check for SCPs affecting RAM operations

MAY:
- Check AWS Health Dashboard for RAM service issues
- Review application logs for access errors

## Phase 2 — Remediate

MUST:
- Based on triage findings, pivot to the appropriate specific runbook:
  - Sharing issues → A1 (creation), A2 (invitation), A3 (acceptance)
  - Resource issues → B1 (supported), B2 (failures), B3 (cross-account)
  - Organization → C1 (org sharing), C2 (auto-sharing)
  - Permissions → D1 (managed), D2 (custom)
  - Access issues → E1 (access denied), E2 (not visible)
- Document findings and root cause

SHOULD:
- Check for multiple concurrent issues
- Review recent changes that may have caused the issue

MAY:
- Set up comprehensive monitoring
- Create a post-incident report

## Common Issues

- symptoms: "RAM operations failing"
  diagnosis: "Multiple possible causes: permissions, org setup, resource type."
  resolution: "Follow triage steps. Check permissions → org status → resource type → share status."

- symptoms: "Sharing was working but stopped"
  diagnosis: "Resource share deleted, principal removed, or permission changed."
  resolution: "Check CloudTrail for recent changes. Verify share and association status."

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: shares
    content: "<resource share status>"
  - type: invitations
    content: "<invitation status>"
  - type: permissions
    content: "<permission configuration>"
severity: MEDIUM
general_analysis:
  org_sharing: "enabled | disabled"
  shares_status: "active | issues found"
  specific_runbook: "<recommended runbook ID>"
mitigation:
  immediate: "Follow the identified specific runbook"
  long_term: "Implement comprehensive RAM monitoring"
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

- Resource share configurations may contain account IDs and organizational structure
- Principal ARNs reveal cross-account relationships
- Permission policies may expose internal access patterns
- Classification: INTERNAL - contains organizational topology data

## Prohibited Actions

- NEVER modify or delete resource shares in production without owner approval
- NEVER change RAM permissions that could revoke cross-account access to shared resources
- NEVER disable organization sharing without coordinating with all member accounts
- NEVER remove principals from active resource shares without impact analysis

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
