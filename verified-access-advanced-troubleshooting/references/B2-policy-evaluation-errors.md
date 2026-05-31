---
title: "B2 — Policy Evaluation Errors"
description: "Diagnose Cedar policy evaluation failures and unexpected denials"
status: active
severity: HIGH
triggers: ["access denied unexpectedly", "policy evaluation failed", "context data missing", "permit not evaluated"]
owner: devops-agent
objective: "Resolve unexpected policy evaluation results in Verified Access"
context: "Policy evaluation depends on trust context from all attached providers. Missing context, incorrect data types, or evaluation order issues cause unexpected denials."
---

## Phase 1 — Triage

MUST:
- Check access logs for denial details: `aws logs filter-log-events --log-group-name <va-log-group> --filter-pattern '{ $.activity = "denied" }' --start-time <epoch-ms>`
- Get group policy: `aws ec2 get-verified-access-group-policy --verified-access-group-id <group-id>`
- Verify trust providers are attached and configured: `aws ec2 describe-verified-access-trust-providers`

SHOULD:
- Compare access log trust context with policy expectations
- Check if the user's OIDC token contains the expected claims
- Verify device trust agent is reporting correct posture data

MAY:
- Test with different users to isolate user-specific vs policy-wide issues
- Review OIDC provider token configuration for claim mapping

## Phase 2 — Remediate

MUST:
- Ensure trust context data matches Cedar policy expectations
- Fix context path references if they don't match actual data structure
- Verify all referenced trust providers are available and returning data

SHOULD:
- Add logging for specific denial scenarios
- Test policy changes with a subset of users before broad deployment

MAY:
- Implement Cedar policy unit testing
- Create monitoring for policy evaluation failure rates

## Common Issues

- symptoms: "Access denied with 'context data not available' in logs"
  diagnosis: "Trust provider not returning expected context data or provider is unavailable."
  resolution: "Verify trust provider status. Check OIDC token claims. Ensure device trust agent is enrolled and reporting."

## Safety Ratings

```
safety_ratings:
  - "logs filter-log-events: GREEN — read-only log analysis"
  - "get-verified-access-group-policy: GREEN — read-only policy retrieval"
  - "describe-verified-access-trust-providers: GREEN — read-only enumeration"
  - "modify-verified-access-group-policy: YELLOW — changes authorization rules"
```

## Escalation Conditions

- Widespread access denials affecting multiple users or groups
- Trust provider returning incorrect or stale context data
- Policy evaluation results are inconsistent across requests
- Cannot determine denial reason from access logs

## Data Sensitivity

- Access logs: HIGH — contain user identity, device posture, and access decisions
- OIDC token claims: HIGH — contain personal identity information
- Device trust data: HIGH — reveals device security posture
- Cedar policy content: MEDIUM — reveals authorization logic

## Prohibited Actions

- NEVER bypass policy evaluation by disabling trust providers
- NEVER share access log entries containing user PII without authorization
- NEVER modify policies to permit all access as a permanent fix
- NEVER ignore repeated policy evaluation failures without investigation

## Phase 3 — Rollback

1. Revert Cedar policy to last known working version
2. If trust provider was modified, restore previous configuration
3. Verify access is restored for affected users
4. Check access logs to confirm evaluation results are correct
5. Document the root cause and policy fix for future reference

## Output Format

```yaml
root_cause: "policy_evaluation — <specific_cause>"
evidence:
  - type: evaluation_result
    content: "<access log entries and context data>"
severity: HIGH
mitigation:
  immediate: "Fix policy or trust context issue"
  long_term: "Implement policy testing and monitoring"
```

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
