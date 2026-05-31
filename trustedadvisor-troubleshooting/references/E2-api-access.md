---
title: "E2 — API Access"
description: "Diagnose Trusted Advisor API access and permission issues"
status: active
severity: MEDIUM
triggers:
  - "TA API error"
  - "cannot access TA API"
  - "support API permission"
  - "API access denied"
owner: devops-agent
objective: "Resolve Trusted Advisor API access and permission issues"
context: "The Trusted Advisor API is part of the AWS Support API. It requires Business/Enterprise Support, us-east-1 endpoint, and IAM permissions for support:describeTrustedAdvisor* and support:refreshTrustedAdvisorCheck actions."
---

## Phase 1 — Triage

MUST:
- Verify API endpoint is us-east-1: ensure --region us-east-1 is specified
- Check IAM permissions: `aws iam simulate-principal-policy --policy-source-arn <arn> --action-names support:DescribeTrustedAdvisorChecks --region us-east-1`
- Verify support plan allows API access
- Test basic API call: `aws support describe-trusted-advisor-checks --language en --region us-east-1`

SHOULD:
- Check for SCP blocking support:* actions
- Verify no permission boundary restricting support API
- Review CloudTrail for authorization failures

MAY:
- Check if API rate limits are being hit
- Verify SDK/CLI version supports the API

## Phase 2 — Remediate

MUST:
- Add IAM permissions for support:describeTrustedAdvisor* and support:refreshTrustedAdvisorCheck
- Use --region us-east-1 for all API calls
- Ensure Business or Enterprise Support plan

SHOULD:
- Create dedicated IAM policy for Trusted Advisor access
- Remove SCP restrictions if blocking legitimate access
- Implement proper error handling in API automation

MAY:
- Set up IAM role for cross-account TA API access
- Implement API call caching to reduce rate limit impact

## Common Issues

- symptoms: "Could not connect to the endpoint URL"
  diagnosis: "API call not targeting us-east-1."
  resolution: "Add --region us-east-1 to all Trusted Advisor API calls."

- symptoms: "AccessDeniedException"
  diagnosis: "IAM policy missing support:* permissions."
  resolution: "Add support:DescribeTrustedAdvisor* and support:RefreshTrustedAdvisorCheck to IAM policy."

- symptoms: "API returns empty check list"
  diagnosis: "Support plan is Basic/Developer (limited checks) or permissions issue."
  resolution: "Verify support plan. Check IAM permissions."

## Output Format

```yaml
root_cause: "api_access — <specific_cause>"
evidence:
  - type: api_error
    content: "<specific API error message>"
  - type: permissions
    content: "<IAM policy and support plan>"
severity: MEDIUM
mitigation:
  immediate: "Fix region, permissions, or support plan"
  long_term: "Implement dedicated IAM policy and error handling"
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
