---
title: "C1 — Priority Recommendations"
description: "Diagnose Trusted Advisor Priority recommendation issues"
status: active
severity: MEDIUM
triggers:
  - "priority recommendations"
  - "TA priority"
  - "TAM recommendations"
  - "priority not available"
owner: devops-agent
objective: "Resolve Trusted Advisor Priority access and recommendation issues"
context: "Trusted Advisor Priority provides curated, prioritized recommendations through your TAM. It requires Enterprise Support with an assigned Technical Account Manager. Priority recommendations are tailored to your specific environment."
---

## Phase 1 — Triage

MUST:
- Verify support plan is Enterprise: `aws support describe-services --region us-east-1`
- Check if Trusted Advisor Priority is enabled for the account
- Verify TAM assignment (requires AWS Support console or TAM contact)
- Check if priority recommendations are visible in the console

SHOULD:
- Review current priority recommendation status
- Check if recommendations have been acknowledged or resolved
- Verify organizational setup if using Priority across accounts

MAY:
- Review priority recommendation history
- Check if specific recommendation categories are configured

## Phase 2 — Remediate

MUST:
- Confirm Enterprise Support plan with assigned TAM
- Contact TAM to enable Priority if not yet activated
- Review and acknowledge priority recommendations

SHOULD:
- Set up regular review cadence for priority recommendations
- Assign owners to priority recommendations
- Track recommendation resolution progress

MAY:
- Integrate priority recommendations with ticketing systems
- Set up automated notification for new priority recommendations

## Common Issues

- symptoms: "Priority recommendations not available"
  diagnosis: "Account does not have Enterprise Support or no TAM assigned."
  resolution: "Upgrade to Enterprise Support and request TAM assignment."

- symptoms: "Priority recommendations seem generic"
  diagnosis: "Priority needs time to learn your environment patterns."
  resolution: "Provide feedback to TAM. Priority improves with engagement."

- symptoms: "Cannot access Priority across organization"
  diagnosis: "Priority organizational view requires specific setup."
  resolution: "Work with TAM to configure organizational Priority access."

## Output Format

```yaml
root_cause: "priority_recommendations — <specific_cause>"
evidence:
  - type: support_plan
    content: "<support plan and TAM status>"
  - type: priority_status
    content: "<priority recommendation availability>"
severity: MEDIUM
mitigation:
  immediate: "Verify Enterprise Support and TAM assignment"
  long_term: "Establish regular priority recommendation review process"
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
