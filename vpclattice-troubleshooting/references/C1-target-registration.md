---
title: "C1 — Target Registration Issues"
description: "Diagnose target registration and deregistration problems"
status: active
severity: HIGH
triggers:
  - "target not registered"
  - "cannot add target"
  - "target registration failed"
  - "deregister target"
owner: devops-agent
objective: "Resolve target registration issues in VPC Lattice target groups"
context: "Targets are registered in target groups. Target types include IP, Instance, Lambda, and ALB. Each type has specific registration requirements. Target type is immutable after target group creation."
---

## Phase 1 — Triage

MUST:
- List targets in group: `aws vpc-lattice list-targets --target-group-identifier <id>`
- Check target group config: `aws vpc-lattice get-target-group --target-group-identifier <id>`
- Verify target type matches registration (IP vs Instance vs Lambda vs ALB)
- Check target status and reason code

SHOULD:
- For IP targets: verify IP is reachable from VPC Lattice
- For Instance targets: verify instance exists and is running
- For Lambda targets: verify function exists and has resource policy
- For ALB targets: verify ALB is in associated VPC

## Phase 2 — Remediate

MUST:
- Register targets matching the target group type
- For Lambda: add VPC Lattice invoke permission to function resource policy
- Ensure targets are in VPCs associated with the service network

SHOULD:
- Use `register-targets` with correct target identifier format
- Monitor target health after registration

## Output Format

```yaml
root_cause: "target-registration — <specific_cause>"
evidence:
  - type: target_group_type
    content: "<IP|INSTANCE|LAMBDA|ALB>"
  - type: target_status
    content: "<target health and reason>"
severity: HIGH
mitigation:
  immediate: "Fix target registration"
  long_term: "Automate target management"
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
