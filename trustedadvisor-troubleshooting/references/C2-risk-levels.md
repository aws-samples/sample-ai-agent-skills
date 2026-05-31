---
title: "C2 — Risk Levels"
description: "Diagnose risk level interpretation and configuration issues"
status: active
severity: LOW
triggers:
  - "risk level"
  - "check severity"
  - "warning vs error"
  - "check status meaning"
owner: devops-agent
objective: "Clarify risk level interpretation and prioritization"
context: "Trusted Advisor uses three status levels: ok (green), warning (yellow), and error (red). These indicate the severity of findings. Check summaries show counts of resources in each status. Prioritize error (red) findings first."
---

## Phase 1 — Triage

MUST:
- Get check summaries with status counts: `aws support describe-trusted-advisor-check-summaries --check-ids <check-ids> --region us-east-1`
- Identify checks with error (red) status
- Get detailed results for error checks: `aws support describe-trusted-advisor-check-result --check-id <check-id> --language en --region us-east-1`
- Prioritize by category: security errors > performance errors > cost warnings

SHOULD:
- Count total resources in each status across all checks
- Identify trends (increasing warnings may become errors)
- Check if excluded resources are masking true risk

MAY:
- Create risk dashboard from check summaries
- Compare risk levels across accounts in organization

## Phase 2 — Remediate

MUST:
- Address all error (red) findings first
- Review warning (yellow) findings for potential escalation
- Document accepted risks for ok-but-excluded resources

SHOULD:
- Set up EventBridge rules for status changes to error
- Create remediation runbooks for common findings
- Establish SLA for addressing each risk level

MAY:
- Implement automated remediation for specific finding types
- Create organizational risk scorecards

## Common Issues

- symptoms: "Confused about warning vs error severity"
  diagnosis: "Warning (yellow) indicates potential issue. Error (red) indicates confirmed risk."
  resolution: "Prioritize errors. Investigate warnings. Green means no issues found."

- symptoms: "Check shows ok but resources were excluded"
  diagnosis: "Excluded resources are hidden from status counts."
  resolution: "Review excluded resources. Exclusion hides, not fixes, issues."

- symptoms: "Risk level changed without resource changes"
  diagnosis: "Check thresholds or evaluation criteria may have been updated by AWS."
  resolution: "Review check description for current criteria. Refresh and re-evaluate."

## Output Format

```yaml
root_cause: "risk_levels — <specific_cause>"
evidence:
  - type: check_summaries
    content: "<status counts across checks>"
  - type: error_details
    content: "<error-level finding details>"
severity: LOW
mitigation:
  immediate: "Address error findings and review warnings"
  long_term: "Implement risk monitoring and automated remediation"
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
