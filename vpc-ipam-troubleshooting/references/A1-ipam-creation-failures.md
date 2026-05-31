---
title: "A1 — IPAM Creation Failures"
description: "Diagnose VPC IPAM creation failures"
status: active
severity: HIGH
triggers: ["cannot create IPAM", "IPAM creation failed", "create-ipam error"]
owner: devops-agent
objective: "Resolve IPAM creation issues"
context: "IPAM is an organization-level resource. Only one IPAM is typically needed per organization."
---

## Phase 1 — Triage

MUST:
- Check existing IPAMs: `aws ec2 describe-ipams --query 'Ipams[*].{IpamId:IpamId,State:State,OperatingRegions:OperatingRegions}'`
- Verify IAM permissions for ec2:CreateIpam
- Check if an IPAM already exists in the account

SHOULD:
- Verify operating regions are valid AWS regions
- Check for service quotas: `aws service-quotas get-service-quota --service-code vpc --quota-code L-F8B4A9E6`

MAY:
- Review CloudTrail for CreateIpam errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateIpam --max-results 10`

## Phase 2 — Remediate

MUST:
- Fix IAM permissions (ec2:CreateIpam, ec2:CreateTags)
- Specify valid operating regions during creation
- Ensure the account has not exceeded IPAM service quotas

SHOULD:
- Create IPAM from the organization management account or delegated IPAM admin
- Include all regions where VPCs will be managed in operating regions

MAY:
- Use CloudFormation or Terraform for IPAM creation
- Tag IPAM with environment and owner metadata

## Common Issues

- symptoms: "CreateIpam returns UnauthorizedOperation"
  diagnosis: "Caller lacks ec2:CreateIpam permission."
  resolution: "Add ec2:CreateIpam and ec2:CreateTags permissions to the caller's IAM role."

- symptoms: "CreateIpam returns InvalidParameterValue for operating regions"
  diagnosis: "One or more specified operating regions are invalid or not enabled in the account."
  resolution: "Verify all operating regions are valid AWS region codes and enabled in the account."

## Safety Ratings

```
safety_ratings:
  - "describe-ipams: GREEN — read-only IPAM enumeration"
  - "CreateIpam: YELLOW — creates organization-level resource"
  - "service-quotas get-service-quota: GREEN — read-only quota check"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
```

## Escalation Conditions

- IPAM creation fails with internal service errors after fixing permissions
- Organization requires IPAM but management account access is restricted
- IPAM creation blocked by SCP
- Multiple teams need separate IPAMs but quotas are limited

## Data Sensitivity

- `aws ec2 describe-ipams`: LOW — returns IPAM metadata
- `aws service-quotas get-service-quota`: LOW — returns quota information
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history and principals
- IPAM operating regions: LOW — reveals deployment regions

## Prohibited Actions

- NEVER create multiple IPAMs without understanding the organizational IP management strategy
- NEVER create IPAM without including all required operating regions
- NEVER use overly permissive IAM policies (ec2:*) to resolve creation issues
- NEVER skip tagging IPAM resources for cost and ownership tracking

## Phase 3 — Rollback

1. If IPAM was created with incorrect operating regions, modify it: `aws ec2 modify-ipam --ipam-id <ipam-id> --add-operating-regions RegionName=<region>` or `--remove-operating-regions RegionName=<region>`
2. If IPAM must be deleted (no pools or allocations): `aws ec2 delete-ipam --ipam-id <ipam-id>`
3. If IAM permissions were broadened, revert to least-privilege
4. Verify no dependent resources were affected
5. Document the IPAM creation attempt and outcome

## Output Format

```yaml
root_cause: "ipam_creation — <specific_cause>"
evidence:
  - type: creation_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix permissions or configuration"
  long_term: "Deploy IPAM via IaC"
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
