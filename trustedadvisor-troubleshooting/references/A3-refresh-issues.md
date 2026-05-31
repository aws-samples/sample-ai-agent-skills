---
title: "A3 — Refresh Issues"
description: "Diagnose Trusted Advisor check refresh failures"
status: active
severity: MEDIUM
triggers:
  - "refresh failed"
  - "cannot refresh check"
  - "refresh timeout"
  - "refresh error"
owner: devops-agent
objective: "Resolve Trusted Advisor check refresh issues"
context: "Check refresh triggers re-evaluation of resources. Minimum interval is 5 minutes between refreshes. Some checks take longer to refresh than others. Refresh requires appropriate IAM permissions and support plan."
---

## Phase 1 — Triage

MUST:
- Check current refresh status: `aws support describe-trusted-advisor-check-refresh-statuses --check-ids <check-id> --region us-east-1`
- Verify last refresh was more than 5 minutes ago
- Confirm IAM permissions include support:refreshTrustedAdvisorCheck
- Verify support plan supports refresh (Business/Enterprise)

SHOULD:
- Check if the check is currently in progress (status: enqueued or in_progress)
- Verify no rate limiting is occurring
- Check CloudTrail for refresh API errors

MAY:
- Check if multiple checks are being refreshed simultaneously
- Verify account is not under any service restrictions

## Phase 2 — Remediate

MUST:
- Wait at least 5 minutes between refresh requests
- Ensure IAM permissions are correct
- Use --region us-east-1 for refresh API calls

SHOULD:
- Implement refresh scheduling with minimum interval enforcement
- Handle InvalidParameterValueException for too-frequent refreshes
- Monitor refresh completion status

MAY:
- Batch refresh requests with appropriate delays
- Set up automated refresh pipeline with error handling

## Common Issues

- symptoms: "InvalidParameterValueException on refresh"
  diagnosis: "Refresh requested within 5-minute minimum interval."
  resolution: "Wait 5 minutes between refresh requests for the same check."

- symptoms: "Refresh status stuck in enqueued"
  diagnosis: "Check refresh is queued but not yet processing."
  resolution: "Wait for processing. Large checks may take several minutes."

- symptoms: "Refresh succeeds but results unchanged"
  diagnosis: "No resource changes since last refresh, or check evaluates infrequently."
  resolution: "Results are correct — no changes detected. Verify resource state independently."

## Output Format

```yaml
root_cause: "refresh_issue — <specific_cause>"
evidence:
  - type: refresh_status
    content: "<refresh status and timing>"
  - type: permissions
    content: "<IAM permissions for refresh>"
severity: MEDIUM
mitigation:
  immediate: "Fix timing, permissions, or wait for completion"
  long_term: "Implement automated refresh with proper interval management"
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
