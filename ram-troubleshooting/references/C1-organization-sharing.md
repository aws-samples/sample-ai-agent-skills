---
title: "C1 — Organization Sharing Issues"
description: "Diagnose RAM organization sharing configuration problems"
status: active
severity: HIGH
triggers:
  - "organization sharing not working"
  - "org sharing failed"
  - "enable sharing with organization"
  - "organization principal error"
owner: devops-agent
objective: "Identify and resolve RAM organization sharing issues"
context: "RAM organization sharing allows automatic resource sharing with member accounts without invitations. It requires explicit enablement from the management account. Once enabled, resources shared with organization or OU principals are automatically available to member accounts."
---

## Phase 1 — Triage

MUST:
- Verify Organizations is enabled: `aws organizations describe-organization`
- Check if RAM org sharing is enabled (try sharing with org principal)
- Verify the caller is in the management account for enablement
- Check for the RAM service-linked role: `aws iam get-role --role-name AWSServiceRoleForResourceAccessManager`

SHOULD:
- Check CloudTrail for EnableSharingWithAwsOrganization events
- Verify the organization ARN format is correct
- Check for SCPs blocking RAM operations
- Verify member accounts are active in the organization

MAY:
- Check if specific OUs are correctly targeted
- Review organization structure for sharing scope

## Phase 2 — Remediate

MUST:
- Enable organization sharing from management account: `aws ram enable-sharing-with-aws-organization`
- Verify the service-linked role is created
- Use correct organization/OU ARN format for principals

SHOULD:
- Test sharing with a single resource and OU
- Document the organization sharing setup
- Verify member accounts can access shared resources

MAY:
- Set up monitoring for organization sharing status
- Create automation for organization-wide sharing

## Common Issues

- symptoms: "Cannot share with organization principal"
  diagnosis: "RAM organization sharing not enabled."
  resolution: "Run: aws ram enable-sharing-with-aws-organization from management account."

- symptoms: "Member account cannot see shared resources"
  diagnosis: "Organization sharing enabled but resource share not using org/OU principal."
  resolution: "Add organization or OU ARN as principal to the resource share."

## Output Format

```yaml
root_cause: "organization_sharing — <specific_cause>"
evidence:
  - type: org_status
    content: "<Organizations and RAM sharing status>"
  - type: share_config
    content: "<resource share principal configuration>"
severity: HIGH
mitigation:
  immediate: "Enable organization sharing and fix principal configuration"
  long_term: "Standardize organization sharing patterns and automate setup"
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

- Resource share configurations may contain account IDs and organizational structure
- Principal ARNs reveal cross-account relationships
- Permission policies may expose internal access patterns
- Classification: INTERNAL - contains organizational topology data

## Prohibited Actions

- NEVER modify or delete resource shares in production without owner approval
- NEVER change RAM permissions that could revoke cross-account access to shared resources
- NEVER disable organization sharing without coordinating with all member accounts
- NEVER remove principals from active resource shares without impact analysis

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
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
