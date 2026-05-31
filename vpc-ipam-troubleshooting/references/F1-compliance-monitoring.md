---
title: "F1 — Compliance Monitoring"
description: "Diagnose IPAM compliance monitoring and reporting issues"
status: active
severity: MEDIUM
triggers: ["non-compliant CIDR", "compliance status", "unmanaged VPC", "IPAM compliance"]
owner: devops-agent
objective: "Resolve IPAM compliance monitoring and non-compliance issues"
context: "IPAM compliance checks if VPC CIDRs are allocated from IPAM pools. Manually assigned CIDRs show as non-compliant."
---

## Phase 1 — Triage

MUST:
- Check resource compliance: `aws ec2 get-ipam-resource-cidrs --ipam-scope-id <scope-id> --query 'IpamResourceCidrs[*].{ResourceId:ResourceId,ResourceCidr:ResourceCidr,ComplianceStatus:ComplianceStatus,ManagementState:ManagementState}'`
- Identify non-compliant resources and their CIDRs
- Verify resource discovery is running and up to date

SHOULD:
- Filter for non-compliant resources: `aws ec2 get-ipam-resource-cidrs --ipam-scope-id <scope-id> --filters Name=compliance-status,Values=noncompliant`
- Check if non-compliant CIDRs overlap with IPAM-managed ranges

MAY:
- Generate compliance reports for audit purposes
- Review compliance trends over time

## Phase 2 — Remediate

MUST:
- Determine if non-compliant CIDRs should be migrated to IPAM management
- For new VPCs, enforce IPAM allocation via SCP or organizational policy
- Document accepted non-compliant CIDRs with justification

SHOULD:
- Plan migration of existing VPCs to IPAM-managed CIDRs
- Implement SCP to require IPAM pool allocation for VPC creation

MAY:
- Create compliance dashboards for management visibility
- Set up automated compliance reporting

## Common Issues

- symptoms: "All VPCs show as non-compliant"
  diagnosis: "VPCs were created before IPAM was configured and use manually assigned CIDRs."
  resolution: "This is expected. Plan migration to IPAM-managed CIDRs or document as accepted non-compliance."

- symptoms: "VPC allocated from IPAM pool shows as non-compliant"
  diagnosis: "Resource discovery has not completed its scan or there is a scope mismatch."
  resolution: "Wait for resource discovery to complete. Verify the VPC CIDR was allocated from a pool in the correct scope."

## Safety Ratings

```
safety_ratings:
  - "get-ipam-resource-cidrs: GREEN — read-only compliance listing"
  - "Compliance reporting: GREEN — read-only data aggregation"
  - "SCP implementation: RED — affects organization-wide VPC creation"
  - "VPC CIDR migration: RED — requires workload migration planning"
```

## Escalation Conditions

- Compliance audit requires 100% IPAM management but legacy VPCs exist
- Non-compliant CIDRs overlap with IPAM-managed ranges
- SCP enforcement blocks legitimate VPC creation workflows
- Compliance reporting required for regulatory audit with tight deadline

## Data Sensitivity

- `aws ec2 get-ipam-resource-cidrs`: MEDIUM — reveals IP allocations and compliance status
- Compliance reports: MEDIUM — reveal organizational IP management posture
- Non-compliant resource list: MEDIUM — reveals unmanaged infrastructure
- SCP policies: MEDIUM — reveal organizational security controls

## Prohibited Actions

- NEVER mark non-compliant resources as compliant without actual remediation
- NEVER implement SCPs that block emergency VPC creation
- NEVER delete non-compliant VPCs to improve compliance metrics
- NEVER ignore non-compliant CIDRs that overlap with managed ranges

## Phase 3 — Rollback

1. If SCP was implemented and blocks legitimate operations, revert the SCP
2. If compliance monitoring was misconfigured, verify resource discovery settings
3. Restore any VPC configurations changed during compliance remediation
4. Verify compliance status reflects the actual state after changes
5. Document compliance exceptions and their justification

## Output Format

```yaml
root_cause: "compliance_monitoring — <specific_cause>"
evidence:
  - type: compliance_status
    content: "<non-compliant resources>"
severity: MEDIUM
mitigation:
  immediate: "Document or remediate non-compliance"
  long_term: "Enforce IPAM allocation via SCP"
```

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
