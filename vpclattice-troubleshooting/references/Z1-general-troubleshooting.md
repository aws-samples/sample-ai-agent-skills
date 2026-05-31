---
title: "Z1 — General VPC Lattice Troubleshooting"
description: "Catch-all runbook for VPC Lattice issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers:
  - "VPC Lattice not working"
  - "lattice issue"
  - "lattice error"
  - "something wrong with lattice"
owner: devops-agent
objective: "Systematic triage of unspecified VPC Lattice issues"
context: "Use this runbook when the user reports a VPC Lattice problem without specific symptoms. Walk through the full stack: service network, VPC association, service, listener, target group, auth policy, DNS, and connectivity."
---

## Phase 1 — Triage

MUST:
- List service networks: `aws vpc-lattice list-service-networks`
- List services: `aws vpc-lattice list-services`
- Check VPC associations: `aws vpc-lattice list-service-network-vpc-associations --service-network-identifier <id>`
- Check service associations: `aws vpc-lattice list-service-network-service-associations --service-network-identifier <id>`
- Check target health: `aws vpc-lattice list-targets --target-group-identifier <id>`
- Check auth policies at both levels
- Check access logs if enabled

SHOULD:
- Review CloudTrail for recent changes
- Check CloudWatch metrics for error rates
- Verify DNS resolution from client VPC

## Phase 2 — Route to specific runbook

Based on triage findings, route to:
- A1/A2 for service network issues
- B1/B2 for service/listener issues
- C1/C2 for target group issues
- D1/D2 for rule/routing issues
- E1/E2 for auth policy issues
- F1/F2 for cross-account issues
- G1/G2 for DNS/health check issues
- H1/H2 for logging/VPC association issues

## Output Format

```yaml
root_cause: "general — <routed_to_specific_runbook>"
evidence:
  - type: triage_summary
    content: "<findings from systematic check>"
severity: MEDIUM
mitigation:
  immediate: "Follow specific runbook for identified issue"
  long_term: "Implement monitoring for all VPC Lattice components"
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
