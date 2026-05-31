---
title: "B1 — Policy Configuration"
description: "Diagnose resiliency policy configuration issues"
status: active
severity: HIGH
triggers: ["policy error", "RTO RPO config", "policy not valid"]
owner: devops-agent
objective: "Resolve resiliency policy configuration issues"
context: "Resiliency policies define RTO/RPO targets per disruption type. Misconfigured policies lead to incorrect compliance assessments."
---

## Phase 1 — Triage

MUST:
- List policies: `aws resiliencehub list-resiliency-policies`
- Check policy details: `aws resiliencehub describe-resiliency-policy --policy-arn <arn>`
- Verify RTO/RPO values for each disruption type (Software, Hardware, AZ, Region)

SHOULD:
- Verify policy tier matches application criticality
- Check if policy values are realistic for the architecture

MAY:
- Compare with AWS recommended values for the tier

## Phase 2 — Remediate

MUST:
- Set appropriate RTO/RPO for each disruption type
- Ensure values are achievable with current architecture

SHOULD:
- Use predefined policy tiers as starting points
- Document rationale for custom RTO/RPO values

MAY:
- Create separate policies for different application tiers

## Common Issues

- symptoms: "All components show non-compliant for Region disruption"
  diagnosis: "Region RTO/RPO set to very low values but app is single-region."
  resolution: "Adjust Region RTO/RPO to realistic values or implement cross-region architecture."

## Output Format

```yaml
root_cause: "policy_configuration — <specific_cause>"
evidence:
  - type: policy_details
    content: "<RTO/RPO values per disruption type>"
severity: HIGH
mitigation:
  immediate: "Fix policy configuration"
  long_term: "Align policies with architecture capabilities"
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

- Application architecture details reveal infrastructure topology
- Assessment results may expose security weaknesses
- RTO/RPO configurations reveal business continuity priorities
- Classification: INTERNAL - contains infrastructure architecture details

## Prohibited Actions

- NEVER delete resiliency policies attached to production applications
- NEVER modify RTO/RPO targets without stakeholder approval
- NEVER dismiss assessment recommendations without documented justification
- NEVER remove application resources from resilience assessments without review

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
