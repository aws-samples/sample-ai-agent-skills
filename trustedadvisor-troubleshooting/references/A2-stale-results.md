---
title: "A2 — Stale Results"
description: "Diagnose stale or outdated Trusted Advisor check results"
status: active
severity: MEDIUM
triggers:
  - "stale results"
  - "outdated check"
  - "old recommendations"
  - "results not current"
owner: devops-agent
objective: "Identify and resolve stale Trusted Advisor check results"
context: "Trusted Advisor results are point-in-time snapshots. Some checks auto-refresh weekly, others need manual refresh. Results can be days old. Always verify the last refresh timestamp before acting on results."
---

## Phase 1 — Triage

MUST:
- Check refresh status: `aws support describe-trusted-advisor-check-refresh-statuses --check-ids <check-id> --region us-east-1`
- Get check result with timestamp: `aws support describe-trusted-advisor-check-result --check-id <check-id> --language en --region us-east-1`
- Verify when the check was last refreshed
- Determine if the check supports manual refresh

SHOULD:
- Check if auto-refresh is enabled for the check
- Compare result timestamp with recent resource changes
- Verify the check is not currently refreshing

MAY:
- Review refresh history for the check
- Check if other checks are also stale

## Phase 2 — Remediate

MUST:
- Trigger manual refresh: `aws support refresh-trusted-advisor-check --check-id <check-id> --region us-east-1`
- Wait for refresh to complete (check status periodically)
- Verify results are updated after refresh

SHOULD:
- Set up automated refresh schedule for critical checks
- Use EventBridge to trigger actions on check status changes
- Document expected refresh intervals for each check

MAY:
- Implement Lambda-based automated refresh for important checks
- Set up monitoring for check freshness

## Common Issues

- symptoms: "Check shows resources that no longer exist"
  diagnosis: "Check results are stale and haven't been refreshed since resource deletion."
  resolution: "Trigger manual refresh. Wait for completion. Verify updated results."

- symptoms: "Cannot refresh — too soon"
  diagnosis: "Minimum 5-minute interval between refreshes not met."
  resolution: "Wait 5 minutes between refresh requests."

- symptoms: "Check always shows same results"
  diagnosis: "Auto-refresh may be weekly. Manual refresh not triggered."
  resolution: "Trigger manual refresh. Set up automated refresh schedule."

## Output Format

```yaml
root_cause: "stale_results — <specific_cause>"
evidence:
  - type: refresh_status
    content: "<last refresh time and status>"
  - type: check_result
    content: "<current check results>"
severity: MEDIUM
mitigation:
  immediate: "Trigger manual refresh and verify results"
  long_term: "Implement automated refresh schedule"
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
