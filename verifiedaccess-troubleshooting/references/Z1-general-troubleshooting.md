---
title: "Z1 — General Verified Access Troubleshooting"
description: "Catch-all systematic investigation for Verified Access issues"
status: active
severity: MEDIUM
triggers:
  - "Verified Access issue"
  - "Verified Access problem"
  - "Verified Access error"
  - "general troubleshooting"
owner: devops-agent
objective: "Systematically investigate Verified Access issues using a structured diagnostic approach"
context: "This runbook provides a general-purpose investigation framework for Verified Access issues that don't clearly match a specific runbook. Use this as a starting point, then pivot to a specific runbook once the problem area is identified."
---

## Phase 1 — Triage

MUST:
- Check instances: `aws ec2 describe-verified-access-instances`
- Check groups: `aws ec2 describe-verified-access-groups`
- Check endpoints: `aws ec2 describe-verified-access-endpoints`
- Check trust providers: `aws ec2 describe-verified-access-trust-providers`
- Check logging: `aws ec2 describe-verified-access-instance-logging-configurations --verified-access-instance-ids <id>`

SHOULD:
- Check CloudTrail for recent changes
- Review access logs if logging is enabled
- Verify DNS configuration for endpoints

MAY:
- Check AWS Health Dashboard
- Review recent configuration changes

## Phase 2 — Remediate

MUST:
- Based on triage, pivot to specific runbook:
  - Instance → A1 (creation), A2 (trust provider)
  - Group → B1 (policies), B2 (evaluation)
  - Endpoint → C1 (creation), C2 (DNS), C3 (connectivity)
  - Trust → D1 (identity), D2 (device)
  - Policy → E1 (Cedar), E2 (debugging)
  - Logging → F1 (access logs), F2 (CloudWatch)
  - Networking → G1 (VPC), G2 (security groups)

SHOULD:
- Enable logging if not already enabled
- Document findings

MAY:
- Set up comprehensive monitoring
- Create runbook for recurring issues

## Common Issues

- symptoms: "Verified Access not working — no specific error"
  diagnosis: "Multiple possible causes across instance, policy, endpoint, or network."
  resolution: "Follow triage: instance → trust providers → groups → endpoints → policies."

- symptoms: "Access was working but stopped"
  diagnosis: "Recent change to policy, trust provider, or network configuration."
  resolution: "Check CloudTrail for changes. Review access logs."

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: instance_status
    content: "<instance and trust provider status>"
  - type: endpoint_status
    content: "<endpoint states>"
  - type: policy_status
    content: "<group policy configuration>"
severity: MEDIUM
general_analysis:
  specific_runbook: "<recommended runbook ID>"
mitigation:
  immediate: "Follow the identified specific runbook"
  long_term: "Implement comprehensive monitoring and logging"
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
