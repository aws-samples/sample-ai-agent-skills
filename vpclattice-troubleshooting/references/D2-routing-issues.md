---
title: "D2 — Traffic Routing Issues"
description: "Diagnose weighted routing and traffic splitting problems"
status: active
severity: MEDIUM
triggers:
  - "traffic not splitting"
  - "weighted routing"
  - "canary deployment"
  - "blue green routing"
owner: devops-agent
objective: "Resolve traffic routing and splitting issues"
context: "VPC Lattice supports weighted target groups in listener rules for traffic splitting. Weights determine the proportion of traffic each target group receives. This enables blue/green and canary deployments."
---

## Phase 1 — Triage

MUST:
- Get rule with weighted targets: `aws vpc-lattice get-rule --service-identifier <svc-id> --listener-identifier <lid> --rule-identifier <rid>`
- Check weight distribution in rule action
- Verify all target groups in the weighted action have healthy targets
- Check access logs for actual traffic distribution

SHOULD:
- Verify weights sum correctly (relative proportions)
- Check if one target group has all unhealthy targets (gets no traffic)

## Phase 2 — Remediate

MUST:
- Ensure all target groups in weighted action have healthy targets
- Set weights to desired proportions (e.g., 90/10 for canary)
- Monitor traffic distribution via access logs or CloudWatch metrics

SHOULD:
- Use gradual weight shifts for safe deployments
- Implement rollback plan if new target group shows errors

## Output Format

```yaml
root_cause: "routing — <specific_cause>"
evidence:
  - type: weight_config
    content: "<target group weights>"
  - type: target_health
    content: "<health per target group>"
severity: MEDIUM
mitigation:
  immediate: "Adjust weights or fix unhealthy targets"
  long_term: "Automate canary deployments with weight management"
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

- Service network configurations reveal microservice architecture
- Auth policies contain inter-service access control rules
- Access logs reveal service-to-service communication patterns
- Classification: INTERNAL - service mesh topology and access patterns

## Prohibited Actions

- NEVER delete service networks with active service associations
- NEVER modify auth policies that could disrupt inter-service communication
- NEVER remove target groups serving production traffic
- NEVER change listener rules without testing routing behavior first

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
  - command: "describe-security-groups"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-network-acls"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-route-tables"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound rules as a fix"
  - "NEVER suggest disabling NACLs to troubleshoot"
  - "NEVER remove all route table entries"
