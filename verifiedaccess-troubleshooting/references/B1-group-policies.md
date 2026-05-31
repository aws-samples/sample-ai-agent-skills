---
title: "B1 — Group Policy Issues"
description: "Diagnose Verified Access group policy configuration problems"
status: active
severity: HIGH
triggers:
  - "group policy error"
  - "policy not working"
  - "access denied by policy"
  - "Cedar policy error"
owner: devops-agent
objective: "Identify and resolve group policy configuration issues"
context: "Cedar policies are attached to Verified Access groups. All endpoints in a group share the same policy. Policies use permit/forbid statements with when clauses that evaluate trust provider context. An empty policy denies all access. Policy syntax errors prevent policy from being applied."
---

## Phase 1 — Triage

MUST:
- Get group policy: `aws ec2 get-verified-access-group-policy --verified-access-group-id <group-id>`
- Verify policy is enabled on the group
- Check Cedar policy syntax for errors
- Verify context paths match trust provider policy reference names

SHOULD:
- Check access logs for policy evaluation results (if logging enabled)
- Verify trust provider is attached to the instance
- Test with a simple permit-all policy to isolate policy issues

MAY:
- Review Cedar policy documentation for syntax reference
- Check if multiple forbid statements are overriding permits

## Phase 2 — Remediate

MUST:
- Fix Cedar policy syntax errors
- Ensure at least one permit statement exists for legitimate access
- Update policy: `aws ec2 modify-verified-access-group-policy --verified-access-group-id <group-id> --policy-enabled --policy-document 'permit(principal, action, resource) when { context.<provider>.<claim> == "value" };'`

SHOULD:
- Test policy changes with a small group of users first
- Enable logging to verify policy evaluation results

MAY:
- Implement graduated policies (start permissive, tighten over time)
- Document policy intent alongside Cedar code

## Common Issues

- symptoms: "All access denied"
  diagnosis: "Policy not enabled, empty policy, or no permit statements."
  resolution: "Enable policy. Add permit statement. Check --policy-enabled flag."

- symptoms: "Policy syntax error on update"
  diagnosis: "Invalid Cedar syntax."
  resolution: "Check Cedar syntax: permit(principal, action, resource) when { <condition> };"

- symptoms: "Policy permits but access still denied"
  diagnosis: "Context path doesn't match trust provider reference name."
  resolution: "Verify context.<policy-reference-name>.<claim> matches trust provider config."

## Output Format

```yaml
root_cause: "group_policy — <specific_cause>"
evidence:
  - type: policy_document
    content: "<Cedar policy>"
  - type: policy_enabled
    content: "<policy enabled status>"
  - type: context_paths
    content: "<trust provider reference names>"
severity: HIGH
mitigation:
  immediate: "Fix Cedar policy syntax and context paths"
  long_term: "Implement policy testing and logging"
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
