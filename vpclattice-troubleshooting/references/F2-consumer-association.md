---
title: "F2 — Consumer VPC Association Issues"
description: "Diagnose cross-account consumer VPC association problems"
status: active
severity: HIGH
triggers:
  - "consumer cannot associate"
  - "VPC association failed cross-account"
  - "shared service network"
  - "consumer VPC"
owner: devops-agent
objective: "Resolve consumer-side VPC association with shared service networks"
context: "After accepting a RAM share, the consumer account must associate their VPC with the shared service network. This enables DNS resolution and connectivity to services in the shared network."
---

## Phase 1 — Triage

MUST:
- Verify RAM share is accepted in consumer account
- Check VPC association: `aws vpc-lattice list-service-network-vpc-associations --service-network-identifier <shared-id>`
- Verify association status is ACTIVE (not CREATE_FAILED or DELETE_IN_PROGRESS)
- Check consumer IAM permissions for `vpc-lattice:CreateServiceNetworkVpcAssociation`

SHOULD:
- Verify consumer VPC has no conflicting CIDR with link-local ranges
- Check if security groups in consumer VPC allow link-local traffic

## Phase 2 — Remediate

MUST:
- Accept RAM share first if not already accepted
- Associate consumer VPC: `aws vpc-lattice create-service-network-vpc-association --service-network-identifier <id> --vpc-identifier <vpc-id>`
- Wait for ACTIVE state before testing connectivity

SHOULD:
- Allow outbound to 169.254.171.0/24 in consumer VPC security groups
- Test DNS resolution from consumer VPC after association

## Output Format

```yaml
root_cause: "consumer-association — <specific_cause>"
evidence:
  - type: ram_share_status
    content: "<accepted or pending>"
  - type: vpc_association
    content: "<association state>"
severity: HIGH
mitigation:
  immediate: "Accept RAM share and create VPC association"
  long_term: "Automate cross-account onboarding"
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
