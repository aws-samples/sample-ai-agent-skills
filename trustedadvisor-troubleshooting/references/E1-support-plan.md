---
title: "E1 — Support Plan Requirements"
description: "Diagnose support plan limitations affecting Trusted Advisor access"
status: active
severity: MEDIUM
triggers:
  - "support plan limitation"
  - "SubscriptionRequiredException"
  - "limited checks"
  - "upgrade support plan"
owner: devops-agent
objective: "Clarify support plan requirements for Trusted Advisor features"
context: "Trusted Advisor features vary by support plan. Basic/Developer: 7 core checks. Business: all checks + API access. Enterprise: all checks + API + Priority. Full API access requires Business or Enterprise."
---

## Phase 1 — Triage

MUST:
- Check current support plan: `aws support describe-services --region us-east-1` (if accessible)
- Identify which TA features are needed
- Verify if SubscriptionRequiredException is occurring
- List available checks to confirm plan level: `aws support describe-trusted-advisor-checks --language en --region us-east-1`

SHOULD:
- Document which checks are available on current plan
- Identify if full API access is required
- Check if Priority features are needed

MAY:
- Compare feature availability across support plans
- Estimate cost of support plan upgrade

## Phase 2 — Remediate

MUST:
- Use available core checks on Basic/Developer plans
- Upgrade to Business Support for full check and API access
- Upgrade to Enterprise for Priority features

SHOULD:
- Document which features require which support plan
- Use console access for checks not available via API on lower plans
- Consider Business Support as minimum for production workloads

MAY:
- Implement alternative monitoring for checks not available on current plan
- Use AWS Config rules as supplement to TA checks

## Common Issues

- symptoms: "SubscriptionRequiredException on API call"
  diagnosis: "Account has Basic or Developer Support plan."
  resolution: "Upgrade to Business Support for full API access."

- symptoms: "Only 7 checks visible"
  diagnosis: "Basic or Developer plan only includes core checks."
  resolution: "Upgrade to Business for all checks. Core checks: SG, S3, IAM, MFA, EBS/RDS snapshots, service limits."

- symptoms: "Priority not available on Enterprise plan"
  diagnosis: "Priority requires Enterprise Support with assigned TAM."
  resolution: "Contact AWS to ensure TAM is assigned. Request Priority enablement."

## Output Format

```yaml
root_cause: "support_plan — <specific_cause>"
evidence:
  - type: plan_level
    content: "<current support plan>"
  - type: available_features
    content: "<features available on current plan>"
severity: MEDIUM
mitigation:
  immediate: "Use available features or upgrade support plan"
  long_term: "Align support plan with operational requirements"
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
