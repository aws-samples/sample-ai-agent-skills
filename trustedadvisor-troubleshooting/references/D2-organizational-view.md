---
title: "D2 — Organizational View"
description: "Diagnose Trusted Advisor organizational view issues"
status: active
severity: MEDIUM
triggers:
  - "organizational view not working"
  - "multi-account TA"
  - "org trusted advisor"
  - "cross-account checks"
owner: devops-agent
objective: "Resolve Trusted Advisor organizational view configuration issues"
context: "Organizational view aggregates Trusted Advisor results across all accounts in an AWS Organization. Requires Organizations with all features enabled and TA enabled as a trusted service. Available from management account or delegated admin."
---

## Phase 1 — Triage

MUST:
- Verify Organizations has all features enabled: `aws organizations describe-organization --query 'Organization.FeatureSet'`
- Check if TA is enabled as trusted service: `aws organizations list-aws-service-access-for-organization --query 'EnabledServicePrincipals[?ServicePrincipal==\`reporting.trustedadvisor.amazonaws.com\`]'`
- Verify calling from management account or delegated admin
- Check support plan on member accounts (Business/Enterprise needed for full checks)

SHOULD:
- Verify delegated administrator if not using management account: `aws organizations list-delegated-administrators --service-principal reporting.trustedadvisor.amazonaws.com`
- Check if organizational view reports are generated
- Verify IAM permissions for organizational view access

MAY:
- Check if specific accounts are excluded from organizational view
- Review organizational view report history

## Phase 2 — Remediate

MUST:
- Enable all features in Organizations (if only consolidated billing)
- Enable TA as trusted service: `aws organizations enable-aws-service-access --service-principal reporting.trustedadvisor.amazonaws.com`
- Access from management account or register delegated admin

SHOULD:
- Register delegated admin for TA: `aws organizations register-delegated-administrator --account-id <id> --service-principal reporting.trustedadvisor.amazonaws.com`
- Ensure member accounts have appropriate support plans
- Generate organizational view reports

MAY:
- Set up automated organizational view report generation
- Create cross-account remediation workflows

## Common Issues

- symptoms: "Organizational view not available"
  diagnosis: "Organizations not using all features or TA not enabled as trusted service."
  resolution: "Enable all features. Enable TA service access in Organizations."

- symptoms: "Some accounts missing from organizational view"
  diagnosis: "Member accounts may have Basic support (limited checks)."
  resolution: "Upgrade member account support plans for full check coverage."

- symptoms: "Cannot access organizational view as delegated admin"
  diagnosis: "Account not registered as delegated administrator for TA."
  resolution: "Register as delegated admin from management account."

## Output Format

```yaml
root_cause: "organizational_view — <specific_cause>"
evidence:
  - type: org_config
    content: "<Organizations feature set and trusted services>"
  - type: access_config
    content: "<management account or delegated admin status>"
severity: MEDIUM
mitigation:
  immediate: "Enable all features and TA trusted service access"
  long_term: "Set up delegated admin and automated reporting"
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

- Check results reveal security vulnerabilities and misconfigurations
- Cost data exposes spending patterns and resource utilization
- Organizational view aggregates findings across all accounts
- Classification: CONFIDENTIAL - security posture and cost intelligence data

## Prohibited Actions

- NEVER dismiss security-critical Trusted Advisor findings without documented justification
- NEVER disable Trusted Advisor checks required by compliance frameworks
- NEVER ignore cost optimization recommendations exceeding defined thresholds without review
- NEVER modify organizational view settings without administrator approval

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
