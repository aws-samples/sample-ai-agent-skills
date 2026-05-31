---
title: "E2 — Policy Debugging"
description: "Debug Verified Access Cedar policy evaluation issues"
status: active
severity: MEDIUM
triggers:
  - "policy debugging"
  - "why access denied"
  - "policy not working as expected"
  - "test policy"
owner: devops-agent
objective: "Debug and resolve Cedar policy evaluation issues"
context: "Policy debugging requires access logs to see evaluation results, context data, and which statements matched. Without logging, debugging is blind. The debugging workflow: enable logging → reproduce issue → analyze logs → fix policy → verify."
---

## Phase 1 — Triage

MUST:
- Enable logging if not already enabled
- Reproduce the access issue
- Check access logs: `aws logs filter-log-events --log-group-name <group> --filter-pattern "{ $.activity = \"DENY\" }"`
- Examine log entries for: policy evaluation result, context data, matched statements

SHOULD:
- Compare expected context values with actual values in logs
- Check if the issue is missing context (trust provider) vs wrong policy logic
- Test with a permit-all policy to confirm non-policy issues

MAY:
- Create a test group with debug policy for isolated testing
- Use Cedar playground for offline policy testing

## Phase 2 — Remediate

MUST:
- Fix policy based on log analysis
- Verify fix with test access
- Confirm logging captures the corrected evaluation

SHOULD:
- Document the debugging process and findings
- Keep logging enabled for ongoing monitoring

MAY:
- Implement automated policy testing
- Create policy templates for common patterns

## Common Issues

- symptoms: "Cannot determine why access is denied"
  diagnosis: "Logging not enabled."
  resolution: "Enable logging first. Reproduce issue. Analyze logs."

- symptoms: "Context data shows null for expected claims"
  diagnosis: "Trust provider not returning expected claims or wrong scope."
  resolution: "Check trust provider scopes. Verify IdP returns the claims."

- symptoms: "Policy works for some users but not others"
  diagnosis: "Different claim values for different users."
  resolution: "Check logs for each user's context. Verify policy conditions match all valid values."

## Output Format

```yaml
root_cause: "policy_debugging — <specific_cause>"
evidence:
  - type: access_logs
    content: "<relevant log entries>"
  - type: context_data
    content: "<actual context values>"
  - type: policy_evaluation
    content: "<evaluation result and matched statements>"
severity: MEDIUM
mitigation:
  immediate: "Fix policy based on log analysis"
  long_term: "Maintain logging and implement policy testing"
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

- Trust provider configurations contain identity provider details
- Access logs reveal user access patterns and device information
- Cedar policies define authorization logic for applications
- Classification: CONFIDENTIAL - access control policies and user activity data

## Prohibited Actions

- NEVER modify trust provider configurations during active user sessions
- NEVER change Verified Access policies that could lock out production users
- NEVER delete endpoints serving active application traffic
- NEVER modify Cedar policies without testing in a non-production environment first

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
