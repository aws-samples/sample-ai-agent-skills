---
title: "D2 — Cost Optimization"
description: "Diagnose cost-related recommendation issues"
status: active
severity: LOW
triggers: ["cost of recommendations", "expensive recommendations", "budget constraints"]
owner: devops-agent
objective: "Balance resilience recommendations with cost constraints"
context: "Recommendations may suggest costly changes. Understanding cost implications helps prioritize implementation."
---

## Phase 1 — Triage

MUST:
- Review cost estimates in recommendations
- Identify which recommendations have highest cost impact
- Check current vs recommended architecture costs

SHOULD:
- Categorize recommendations by cost tier (low/medium/high)
- Identify free or low-cost improvements

MAY:
- Compare costs across different recommendation options

## Phase 2 — Remediate

MUST:
- Implement low-cost, high-impact recommendations first
- Document cost-benefit analysis for expensive recommendations

SHOULD:
- Adjust policy targets if all recommendations exceed budget
- Look for alternative approaches to meet targets at lower cost

MAY:
- Use AWS Pricing Calculator for detailed cost estimates

## Common Issues

- symptoms: "All recommendations require Multi-AZ or cross-region"
  diagnosis: "Policy targets require high availability that needs redundant infrastructure."
  resolution: "Adjust targets to match budget or implement incrementally."

## Output Format

```yaml
root_cause: "cost_optimization — <specific_cause>"
evidence:
  - type: cost_analysis
    content: "<recommendation costs and priorities>"
severity: LOW
mitigation:
  immediate: "Implement low-cost improvements"
  long_term: "Build resilience budget into infrastructure planning"
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
