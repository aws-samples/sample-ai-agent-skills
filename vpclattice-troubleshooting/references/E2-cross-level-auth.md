---
title: "E2 — Cross-Level Auth Policy Conflicts"
description: "Diagnose conflicts between service network and service auth policies"
status: active
severity: HIGH
triggers:
  - "policy conflict"
  - "network policy vs service policy"
  - "dual auth policy"
  - "auth mismatch"
owner: devops-agent
objective: "Resolve auth policy conflicts across levels"
context: "Service network auth policies and service auth policies are evaluated independently. Both must allow. A restrictive service network policy can block access even if the service policy allows it, and vice versa."
---

## Phase 1 — Triage

MUST:
- Get service network auth type and policy: `aws vpc-lattice get-service-network --service-network-identifier <id>`
- Get service auth type and policy: `aws vpc-lattice get-service --service-identifier <id>`
- Compare both policies for conflicting allow/deny statements
- Check access logs for which level denied the request

SHOULD:
- Map out all principals and conditions in both policies
- Verify auth type is consistent (both AWS_IAM or both NONE)

## Phase 2 — Remediate

MUST:
- Align policies so both levels allow intended traffic
- Use service network policy for broad access control (e.g., by VPC)
- Use service policy for fine-grained per-service access control

SHOULD:
- Document the auth model (which level controls what)
- Test cross-level policy interactions before production deployment

## Output Format

```yaml
root_cause: "cross-level-auth — <specific_cause>"
evidence:
  - type: network_policy
    content: "<service network auth policy>"
  - type: service_policy
    content: "<service auth policy>"
  - type: conflict_detail
    content: "<which level denied and why>"
severity: HIGH
mitigation:
  immediate: "Align policies across both levels"
  long_term: "Establish auth policy governance model"
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
