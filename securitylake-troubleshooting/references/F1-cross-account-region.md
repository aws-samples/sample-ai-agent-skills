---
title: "F1 — Cross-Account and Cross-Region Issues"
description: "Diagnose multi-account and multi-region collection problems"
status: active
severity: HIGH
triggers:
  - "cross-account collection"
  - "member account logs"
  - "multi-region setup"
  - "delegated admin"
owner: devops-agent
objective: "Resolve cross-account and cross-region collection issues"
context: "Cross-account collection requires AWS Organizations with a delegated administrator. The delegated admin configures sources for member accounts. Cross-region requires enabling Security Lake in each region."
---

## Phase 1 — Triage

MUST:
- Verify delegated admin: `aws securitylake list-data-lakes` (from management or delegated admin account)
- Check Organizations status: `aws organizations describe-organization`
- Verify member account sources: `aws securitylake get-data-lake-sources --accounts <member-account-id>`
- Check exceptions for member accounts: `aws securitylake list-data-lake-exceptions --regions <region>`

SHOULD:
- Verify no SCP blocking Security Lake in member accounts
- Check if member accounts have Security Lake service-linked role
- Verify regions are enabled in member accounts

## Phase 2 — Remediate

MUST:
- Register delegated admin: `aws securitylake register-data-lake-delegated-administrator --account-id <admin-id>`
- Enable sources for member accounts from delegated admin
- Enable Security Lake in each required region

SHOULD:
- Use auto-enable for new member accounts
- Monitor exceptions across all accounts and regions

## Output Format

```yaml
root_cause: "cross-account-region — <specific_cause>"
evidence:
  - type: delegated_admin
    content: "<admin account status>"
  - type: member_sources
    content: "<source status per account>"
severity: HIGH
mitigation:
  immediate: "Fix delegated admin or member account configuration"
  long_term: "Automate multi-account onboarding"
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

- Security logs contain authentication and authorization events
- Network flow logs may reveal infrastructure topology
- CloudTrail data contains API activity across all services
- Classification: HIGHLY CONFIDENTIAL - aggregated security telemetry requires strict access control

## Prohibited Actions

- NEVER delete or disable Security Lake in production without CISO approval
- NEVER modify subscriber access that could disrupt security monitoring
- NEVER change data retention policies without compliance team review
- NEVER remove log sources that are required for regulatory compliance

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
  - "NEVER suggest disabling security standards"
  - "NEVER suppress findings without investigation"
  - "NEVER remove security integrations"
