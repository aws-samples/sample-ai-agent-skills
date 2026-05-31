---
title: "C2 — Compliance Status"
description: "Diagnose unexpected compliance results"
status: active
severity: MEDIUM
triggers: ["non-compliant", "compliance unexpected", "wrong compliance"]
owner: devops-agent
objective: "Interpret and resolve compliance status issues"
context: "Compliance is evaluated per component per disruption type. Unexpected results often stem from policy misconfiguration or resource grouping."
---

## Phase 1 — Triage

MUST:
- Check component compliance: `aws resiliencehub list-app-component-compliances --assessment-arn <arn>`
- Review which disruption types are non-compliant
- Check policy RTO/RPO values for non-compliant types

SHOULD:
- Verify component grouping is correct
- Check if non-compliance is expected (e.g., single-region app for Region disruption)

MAY:
- Compare with previous assessment results

## Phase 2 — Remediate

MUST:
- For legitimate non-compliance: implement recommended architecture changes
- For policy mismatch: adjust RTO/RPO targets to realistic values

SHOULD:
- Prioritize by disruption type severity
- Review recommendations for each non-compliant component

MAY:
- Accept risk for certain disruption types if cost-prohibitive to fix

## Common Issues

- symptoms: "Everything non-compliant for Region disruption"
  diagnosis: "Single-region application with Region RTO/RPO targets."
  resolution: "Either implement cross-region architecture or adjust Region targets."

## Output Format

```yaml
root_cause: "compliance_status — <specific_cause>"
evidence:
  - type: compliance_details
    content: "<per-component per-disruption compliance>"
severity: MEDIUM
mitigation:
  immediate: "Address highest-priority non-compliance"
  long_term: "Align architecture with resilience requirements"
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
