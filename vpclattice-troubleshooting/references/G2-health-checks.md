---
title: "G2 — Health Check Failures"
description: "Diagnose VPC Lattice target group health check issues"
status: active
severity: HIGH
triggers:
  - "health check failing"
  - "targets unhealthy"
  - "health check timeout"
  - "health check configuration"
owner: devops-agent
objective: "Resolve health check configuration and failure issues"
context: "Health checks verify target availability. Supported for IP, Instance, and ALB targets. NOT supported for Lambda targets. Health check misconfiguration is a top cause of targets showing unhealthy."
---

## Phase 1 — Triage

MUST:
- Get health check config: `aws vpc-lattice get-target-group --target-group-identifier <id>`
- Check target health: `aws vpc-lattice list-targets --target-group-identifier <id>`
- Verify health check path exists on target and returns expected status code
- Check target security groups allow inbound from VPC Lattice

SHOULD:
- Verify health check port matches application port (or override port)
- Check health check protocol matches target protocol
- Test health check endpoint manually: `curl http://<target-ip>:<port>/<health-path>`

## Phase 2 — Remediate

MUST:
- Configure health check path that returns HTTP 200
- Allow inbound health check traffic from VPC Lattice prefix list
- Match health check protocol to target protocol (HTTP or HTTPS)

SHOULD:
- Use dedicated /health endpoint
- Set reasonable thresholds (healthy: 3, unhealthy: 3, interval: 30s)
- Do NOT configure health checks for Lambda target groups

## Output Format

```yaml
root_cause: "health-check — <specific_cause>"
evidence:
  - type: health_config
    content: "<path, port, protocol>"
  - type: target_status
    content: "<healthy/unhealthy counts>"
severity: HIGH
mitigation:
  immediate: "Fix health check path or security groups"
  long_term: "Standardize health check endpoints across services"
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
