---
title: "Z1 — General Troubleshooting"
description: "Catch-all runbook for Trusted Advisor issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers:
  - "trusted advisor problem"
  - "TA issue"
  - "something wrong with trusted advisor"
  - "trusted advisor not working"
owner: devops-agent
objective: "Systematically diagnose unspecified Trusted Advisor issues"
context: "This catch-all runbook provides a systematic approach when the specific issue category is unclear. It covers initial data collection, common checks, and routing to specific runbooks."
---

## Phase 1 — Triage

MUST:
- Verify API access: `aws support describe-trusted-advisor-checks --language en --region us-east-1`
- Get check summaries: `aws support describe-trusted-advisor-check-summaries --check-ids <check-ids> --region us-east-1`
- Verify support plan level
- Check if the issue is access-related or results-related

SHOULD:
- Check refresh statuses for stale data
- Verify IAM permissions
- Review recent check status changes

MAY:
- Check AWS Health Dashboard for TA service events
- Review CloudTrail for TA API activity

## Phase 2 — Remediate

MUST:
- Route to the appropriate specific runbook based on triage findings
- If check issue → A1-A3
- If category issue → B1-B3
- If priority issue → C1-C2
- If integration issue → D1-D2
- If access issue → E1-E2

SHOULD:
- Document findings and route to specific runbook
- Collect all relevant evidence before escalating

MAY:
- Contact AWS Support for TA-specific issues
- Review AWS documentation for recent TA changes

## Common Issues

- symptoms: "Trusted Advisor not working (unspecified)"
  diagnosis: "Multiple possible causes. Systematic triage required."
  resolution: "Follow Phase 1 triage to identify category, then route to specific runbook."

- symptoms: "Cannot access any TA features"
  diagnosis: "Support plan limitation or IAM permission issue."
  resolution: "Check support plan and IAM permissions. Use --region us-east-1."

- symptoms: "TA recommendations seem wrong"
  diagnosis: "Stale results or misinterpretation of check criteria."
  resolution: "Refresh checks. Review check description for current evaluation criteria."

## Output Format

```yaml
root_cause: "general — <routed_to_specific_runbook>"
evidence:
  - type: ta_status
    content: "<check summaries and access status>"
  - type: triage_findings
    content: "<initial diagnostic results>"
severity: MEDIUM
mitigation:
  immediate: "Route to specific runbook based on triage"
  long_term: "Implement comprehensive TA monitoring and automation"
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
