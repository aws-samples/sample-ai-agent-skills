---
title: "E1 — Auth Policy Evaluation Issues"
description: "Diagnose auth policy denials and evaluation problems"
status: active
severity: HIGH
triggers:
  - "access denied"
  - "auth policy deny"
  - "403 forbidden"
  - "authorization failed"
owner: devops-agent
objective: "Resolve auth policy evaluation failures"
context: "VPC Lattice auth policies are evaluated at two levels: service network and service. Both must allow the request. Policies use IAM-like syntax with VPC Lattice-specific condition keys. Auth type must be AWS_IAM for policies to be enforced."
---

## Phase 1 — Triage

MUST:
- Check service network auth policy: `aws vpc-lattice get-auth-policy --resource-identifier <service-network-id>`
- Check service auth policy: `aws vpc-lattice get-auth-policy --resource-identifier <service-id>`
- Verify auth type is AWS_IAM on both resources
- Check access logs for `authDeniedReason` field
- Verify caller is signing requests with SigV4

SHOULD:
- Check condition keys in policies (SourceVpc, ServiceNetworkArn, etc.)
- Verify caller IAM role has `vpc-lattice-svcs:Invoke` permission

## Phase 2 — Remediate

MUST:
- Ensure BOTH service network and service auth policies allow the caller
- Verify auth type is AWS_IAM (not NONE) if policies should be enforced
- Ensure client signs requests with SigV4 when auth type is AWS_IAM

SHOULD:
- Use `vpc-lattice-svcs:SourceVpc` condition for VPC-based access control
- Test policies with minimal allow before adding restrictions

## Output Format

```yaml
root_cause: "auth-policy — <specific_cause>"
evidence:
  - type: service_network_policy
    content: "<policy summary>"
  - type: service_policy
    content: "<policy summary>"
  - type: access_log
    content: "<authDeniedReason>"
severity: HIGH
mitigation:
  immediate: "Fix auth policy to allow legitimate callers"
  long_term: "Implement policy testing before deployment"
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
