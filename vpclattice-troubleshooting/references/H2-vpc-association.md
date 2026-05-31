---
title: "H2 — VPC Association Issues"
description: "Diagnose VPC-to-service-network association problems"
status: active
severity: HIGH
triggers:
  - "VPC association failed"
  - "VPC not associated"
  - "association pending"
  - "association error"
owner: devops-agent
objective: "Resolve VPC association issues with service networks"
context: "VPCs must be associated with a service network to access its services. Each VPC can be associated with one service network. Association enables DNS resolution and connectivity. Association failures block all VPC Lattice access from that VPC."
---

## Phase 1 — Triage

MUST:
- List VPC associations: `aws vpc-lattice list-service-network-vpc-associations --service-network-identifier <id>`
- Check association status (ACTIVE, CREATE_IN_PROGRESS, CREATE_FAILED, DELETE_IN_PROGRESS)
- Verify VPC exists and is in available state
- Check IAM permissions for `vpc-lattice:CreateServiceNetworkVpcAssociation`

SHOULD:
- Verify VPC is not already associated with another service network
- Check if security group specified in association allows required traffic
- Review CloudTrail for association failure details

## Phase 2 — Remediate

MUST:
- Ensure VPC is not associated with another service network (1:1 limit)
- Create association: `aws vpc-lattice create-service-network-vpc-association --service-network-identifier <id> --vpc-identifier <vpc-id>`
- Wait for ACTIVE state before testing

SHOULD:
- Specify security group IDs in association for traffic control
- Verify DNS resolution works after association is ACTIVE

## Output Format

```yaml
root_cause: "vpc-association — <specific_cause>"
evidence:
  - type: association_status
    content: "<current state>"
  - type: vpc_status
    content: "<VPC availability>"
severity: HIGH
mitigation:
  immediate: "Fix association or resolve VPC conflict"
  long_term: "Automate VPC association in account provisioning"
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
