---
title: "A2 — Service Network Configuration"
description: "Diagnose service network configuration and auth type issues"
status: active
severity: MEDIUM
triggers:
  - "service network config"
  - "auth type mismatch"
  - "service network settings"
owner: devops-agent
objective: "Resolve service network configuration problems"
context: "Service networks have auth type (NONE or AWS_IAM) and optional auth policies. Changing auth type affects all associated services. Misconfigured auth type is a common cause of access failures."
---

## Phase 1 — Triage

MUST:
- Get service network details: `aws vpc-lattice get-service-network --service-network-identifier <id>`
- Check auth type: verify authType field (NONE or AWS_IAM)
- Check auth policy: `aws vpc-lattice get-auth-policy --resource-identifier <service-network-id>`
- List associated services: `aws vpc-lattice list-service-network-service-associations --service-network-identifier <id>`

SHOULD:
- Verify auth policy JSON is valid
- Check if auth type was recently changed

## Phase 2 — Remediate

MUST:
- Set auth type to AWS_IAM if auth policies are needed
- Ensure auth policy allows intended principals and VPCs
- Test with `vpc-lattice-svcs:SourceVpc` condition key for VPC-based access

SHOULD:
- Use least-privilege auth policies
- Document auth policy changes

## Output Format

```yaml
root_cause: "service-network-config — <specific_cause>"
evidence:
  - type: auth_type
    content: "<NONE or AWS_IAM>"
  - type: auth_policy
    content: "<policy summary>"
severity: MEDIUM
mitigation:
  immediate: "Correct auth type and policy"
  long_term: "Implement auth policy versioning via IaC"
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
