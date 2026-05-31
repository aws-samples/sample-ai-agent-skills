---
title: "A1 — Check Failures"
description: "Diagnose Trusted Advisor check failures and errors"
status: active
severity: HIGH
triggers:
  - "check failed"
  - "check error"
  - "trusted advisor error"
  - "check not available"
owner: devops-agent
objective: "Identify and resolve Trusted Advisor check failures"
context: "Check failures can be caused by insufficient permissions, support plan limitations, API region errors, or service issues. Full API access requires Business/Enterprise Support. API is us-east-1 only."
---

## Phase 1 — Triage

MUST:
- List available checks: `aws support describe-trusted-advisor-checks --language en --region us-east-1`
- Get check result: `aws support describe-trusted-advisor-check-result --check-id <check-id> --language en --region us-east-1`
- Verify support plan level (Business/Enterprise for full access)
- Confirm API calls use us-east-1 region

SHOULD:
- Check for SubscriptionRequiredException (indicates plan limitation)
- Verify IAM permissions include support:describeTrustedAdvisor*
- Check if the specific check is available for the account type

MAY:
- Check AWS Health Dashboard for Trusted Advisor service issues
- Review CloudTrail for API error details

## Phase 2 — Remediate

MUST:
- Use --region us-east-1 for all Trusted Advisor API calls
- Verify support plan supports the requested checks
- Grant IAM permissions for support:* actions

SHOULD:
- Upgrade support plan if full check access is needed
- Use check IDs (not names) for reliable automation
- Handle API errors gracefully in automation scripts

MAY:
- Implement retry logic for transient API failures
- Cache check results to reduce API calls

## Common Issues

- symptoms: "SubscriptionRequiredException"
  diagnosis: "Account has Basic or Developer Support plan."
  resolution: "Upgrade to Business or Enterprise Support for full API access."

- symptoms: "Could not connect to endpoint"
  diagnosis: "API call not targeting us-east-1 region."
  resolution: "Add --region us-east-1 to all Trusted Advisor API calls."

- symptoms: "AccessDeniedException"
  diagnosis: "IAM policy missing support:describeTrustedAdvisor* permissions."
  resolution: "Add support:* or specific Trusted Advisor actions to IAM policy."

## Output Format

```yaml
root_cause: "check_failure — <specific_cause>"
evidence:
  - type: check_result
    content: "<check status and error details>"
  - type: support_plan
    content: "<support plan level>"
severity: HIGH
mitigation:
  immediate: "Fix region, permissions, or support plan"
  long_term: "Implement proper error handling in TA automation"
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
