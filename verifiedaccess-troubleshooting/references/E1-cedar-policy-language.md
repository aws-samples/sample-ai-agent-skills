---
title: "E1 — Cedar Policy Language Issues"
description: "Diagnose Cedar policy syntax and structure problems"
status: active
severity: MEDIUM
triggers:
  - "Cedar syntax error"
  - "policy syntax"
  - "Cedar language"
  - "policy format"
owner: devops-agent
objective: "Identify and resolve Cedar policy language syntax issues"
context: "Cedar is the policy language for Verified Access. It uses permit/forbid statements with when clauses. Cedar is NOT IAM JSON. Common syntax issues include missing semicolons, incorrect operators, wrong context paths, and type mismatches. Cedar is case-sensitive."
---

## Phase 1 — Triage

MUST:
- Get current policy: `aws ec2 get-verified-access-group-policy --verified-access-group-id <group-id>`
- Check for syntax errors in the Cedar policy
- Verify context paths match trust provider policy reference names
- Check operator usage (==, !=, .contains(), .like())

SHOULD:
- Verify string values are quoted
- Check that semicolons terminate each statement
- Verify principal, action, resource placeholders are correct

MAY:
- Reference Cedar language documentation
- Test policy with simplified conditions

## Phase 2 — Remediate

MUST:
- Fix Cedar syntax: `permit(principal, action, resource) when { <condition> };`
- Use correct operators: == for equality, .contains() for set membership, .like() for patterns
- Ensure all statements end with semicolons

SHOULD:
- Update policy: `aws ec2 modify-verified-access-group-policy --verified-access-group-id <group-id> --policy-enabled --policy-document '<cedar-policy>'`
- Test with simple policy first, then add complexity

MAY:
- Use Cedar playground for policy testing
- Implement policy review process

## Common Issues

- symptoms: "Policy update fails with syntax error"
  diagnosis: "Invalid Cedar syntax."
  resolution: "Check: semicolons, quotes around strings, correct operators, matching braces."

- symptoms: "Policy accepted but doesn't match expected behavior"
  diagnosis: "Logic error in when clause or wrong context path."
  resolution: "Verify context paths. Check operator logic. Enable logging to see evaluation."

- symptoms: "String comparison not working"
  diagnosis: "Case sensitivity or missing quotes."
  resolution: "Cedar is case-sensitive. Ensure string values are in double quotes."

## Output Format

```yaml
root_cause: "cedar_syntax — <specific_cause>"
evidence:
  - type: policy_document
    content: "<Cedar policy with error>"
  - type: syntax_error
    content: "<specific syntax issue>"
severity: MEDIUM
mitigation:
  immediate: "Fix Cedar syntax"
  long_term: "Implement policy validation in CI/CD"
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
