---
title: "B2 — Policy Evaluation Issues"
description: "Diagnose unexpected policy evaluation results"
status: active
severity: HIGH
triggers:
  - "policy evaluation"
  - "unexpected deny"
  - "unexpected allow"
  - "policy not matching"
owner: devops-agent
objective: "Identify why policies are not evaluating as expected"
context: "Policy evaluation uses Cedar logic: forbid overrides permit, missing context data results in deny, and context paths must exactly match trust provider reference names. Access logs show evaluation details including which statements matched and what context was available."
---

## Phase 1 — Triage

MUST:
- Enable logging if not already: `aws ec2 modify-verified-access-instance-logging-configuration --verified-access-instance-id <id> --access-logs CloudWatchLogs={Enabled=true,LogGroup=<group>}`
- Check access logs for DENY entries: `aws logs filter-log-events --log-group-name <group> --filter-pattern "DENY"`
- Get current policy: `aws ec2 get-verified-access-group-policy --verified-access-group-id <group-id>`
- Verify trust provider context data available in logs

SHOULD:
- Compare expected context values with actual values in logs
- Check if forbid statements are overriding permits
- Verify user's identity claims match policy conditions

MAY:
- Test with a simplified policy to isolate the issue
- Check if device trust context is missing (device provider not configured)

## Phase 2 — Remediate

MUST:
- Fix context path references to match trust provider policy reference name
- Ensure permit conditions match actual user/device context values
- Remove or fix overly broad forbid statements

SHOULD:
- Add logging to capture all policy evaluation details
- Test policy changes incrementally

MAY:
- Implement policy versioning for rollback capability
- Create test users/devices for policy validation

## Common Issues

- symptoms: "User denied but should be permitted"
  diagnosis: "Context path wrong, claim value mismatch, or forbid overriding permit."
  resolution: "Check logs for actual context values. Fix context paths. Review forbid statements."

- symptoms: "All users denied after policy change"
  diagnosis: "Syntax error in new policy or missing permit statement."
  resolution: "Revert to previous policy. Fix syntax. Ensure permit statement exists."

- symptoms: "Device context missing in evaluation"
  diagnosis: "Device trust provider not attached or device not enrolled."
  resolution: "Verify device trust provider is attached. Ensure user's device is enrolled."

## Output Format

```yaml
root_cause: "policy_evaluation — <specific_cause>"
evidence:
  - type: access_logs
    content: "<policy evaluation log entries>"
  - type: context_data
    content: "<actual context values>"
  - type: policy_document
    content: "<Cedar policy>"
severity: HIGH
mitigation:
  immediate: "Fix policy to match actual context values"
  long_term: "Implement policy testing and monitoring"
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
