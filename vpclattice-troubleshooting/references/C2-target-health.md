---
title: "C2 — Target Health Issues"
description: "Diagnose unhealthy targets and health check failures"
status: active
severity: HIGH
triggers:
  - "target unhealthy"
  - "health check failed"
  - "targets draining"
  - "all targets unhealthy"
owner: devops-agent
objective: "Resolve target health check failures"
context: "Health checks verify target availability. IP and Instance targets support HTTP/HTTPS health checks. ALB targets check ALB health. Lambda targets do NOT support health checks. Unhealthy targets do not receive traffic."
---

## Phase 1 — Triage

MUST:
- Check target health: `aws vpc-lattice list-targets --target-group-identifier <id>`
- Get health check config: `aws vpc-lattice get-target-group --target-group-identifier <id> --query 'config.healthCheck'`
- Verify health check path returns 200 on the target
- Check security groups allow health check traffic from VPC Lattice

SHOULD:
- Test health check endpoint manually from within the VPC
- Verify health check port matches application port
- Check health check protocol matches target protocol

MAY:
- Adjust health check thresholds if targets flap between healthy/unhealthy

## Phase 2 — Remediate

MUST:
- Ensure health check path exists and returns expected status code
- Allow inbound traffic from VPC Lattice managed prefix list
- Match health check protocol to target protocol

SHOULD:
- Use dedicated health check endpoint (e.g., /health)
- Set appropriate healthy/unhealthy thresholds

## Output Format

```yaml
root_cause: "target-health — <specific_cause>"
evidence:
  - type: health_status
    content: "<target health details>"
  - type: health_check_config
    content: "<path, port, protocol, thresholds>"
severity: HIGH
mitigation:
  immediate: "Fix health check configuration or target application"
  long_term: "Implement robust health check endpoints"
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
