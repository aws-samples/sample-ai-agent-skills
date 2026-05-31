---
title: "B1 — Service Creation Issues"
description: "Diagnose VPC Lattice service creation and registration failures"
status: active
severity: HIGH
triggers:
  - "service creation failed"
  - "cannot create lattice service"
  - "service registration error"
owner: devops-agent
objective: "Identify and resolve service creation failures"
context: "A VPC Lattice service represents an application. It requires at least one listener and target group. Services must be associated with a service network to receive traffic."
---

## Phase 1 — Triage

MUST:
- Check IAM permissions for `vpc-lattice:CreateService`
- List existing services: `aws vpc-lattice list-services`
- Get service details if created: `aws vpc-lattice get-service --service-identifier <id>`
- Check CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateService --max-results 5`

SHOULD:
- Verify custom domain name certificate exists in ACM (if using custom domain)
- Check service quota (default: 500 services per region)

## Phase 2 — Remediate

MUST:
- Ensure IAM permissions include `vpc-lattice:CreateService`
- Associate service with service network after creation
- Create at least one listener for the service

SHOULD:
- Configure custom domain with ACM certificate if needed
- Set appropriate auth type on the service

## Output Format

```yaml
root_cause: "service-creation — <specific_cause>"
evidence:
  - type: service_status
    content: "<service state>"
  - type: cloudtrail_event
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix creation blockers"
  long_term: "Automate service creation with IaC"
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
