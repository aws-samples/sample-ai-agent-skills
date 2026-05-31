---
title: "F1 — RAM Cross-Account Sharing Issues"
description: "Diagnose AWS RAM sharing failures for VPC Lattice resources"
status: active
severity: HIGH
triggers:
  - "RAM share failed"
  - "cross-account sharing"
  - "resource share not working"
  - "cannot share service network"
owner: devops-agent
objective: "Resolve RAM-based cross-account sharing issues"
context: "VPC Lattice uses AWS RAM for cross-account sharing of service networks and services. The owner creates a RAM resource share, and the consumer accepts it. Organization-level sharing can auto-accept."
---

## Phase 1 — Triage

MUST:
- List RAM resource shares: `aws ram get-resource-shares --resource-owner SELF`
- Check share status: `aws ram get-resource-share-associations --association-type PRINCIPAL --resource-share-arns <share-arn>`
- Verify shared resource type is `vpc-lattice:ServiceNetwork` or `vpc-lattice:Service`
- Check if consumer account has accepted the share

SHOULD:
- Verify RAM is enabled for the Organization (if using org sharing)
- Check if auto-accept is enabled for the Organization
- Verify principal (account ID or org ID) is correct in the share

## Phase 2 — Remediate

MUST:
- Ensure RAM resource share includes correct VPC Lattice resource ARN
- Add correct consumer account or organization as principal
- Consumer must accept share: `aws ram accept-resource-share-invitation --resource-share-invitation-arn <arn>`

SHOULD:
- Enable auto-accept for Organization-level sharing
- Verify consumer can see shared resources after acceptance

## Output Format

```yaml
root_cause: "ram-sharing — <specific_cause>"
evidence:
  - type: share_status
    content: "<RAM share state>"
  - type: principal_association
    content: "<consumer account association status>"
severity: HIGH
mitigation:
  immediate: "Fix RAM share configuration or accept invitation"
  long_term: "Use Organization-level sharing with auto-accept"
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
