---
title: "A1 — Endpoint Creation Failures"
description: "Diagnose why a VPC interface endpoint creation is failing"
status: active
severity: HIGH
triggers:
  - "endpoint creation failed"
  - "cannot create endpoint"
  - "CreateVpcEndpoint error"
  - "endpoint creation denied"
  - "service not available"
owner: devops-agent
objective: "Identify why interface endpoint creation is failing and resolve the issue"
context: "Interface endpoint creation can fail due to service unavailability in the region, IAM permission issues, VPC limits, subnet configuration problems, duplicate endpoint (one per VPC per service), or endpoint service permission restrictions. The caller needs ec2:CreateVpcEndpoint permission."
---

## Phase 1 — Triage

MUST:
- Check if the service is available in the region: `aws ec2 describe-vpc-endpoint-services --service-names <service-name>`
- Verify IAM permissions: `aws iam simulate-principal-policy --policy-source-arn <caller-arn> --action-names ec2:CreateVpcEndpoint`
- Check if an endpoint already exists for this service in the VPC: `aws ec2 describe-vpc-endpoints --filters Name=vpc-id,Values=<vpc-id> Name=service-name,Values=<service-name>`
- Verify VPC endpoint quota: `aws service-quotas get-service-quota --service-code vpc --quota-code L-29B6F2EB`

SHOULD:
- Verify the specified subnets exist and are in the correct VPC: `aws ec2 describe-subnets --subnet-ids <subnet-ids>`
- Check that only one subnet per AZ is specified
- Verify security group IDs are valid: `aws ec2 describe-security-groups --group-ids <sg-ids>`
- Check CloudTrail for the creation failure event: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateVpcEndpoint --max-results 10`

MAY:
- Check if the VPC has DNS support enabled (required for private DNS)
- Verify the endpoint service allows your account (for custom services)

## Phase 2 — Remediate

MUST:
- Use the correct service name format: com.amazonaws.<region>.<service>
- Ensure only one endpoint per VPC per service
- Fix IAM permissions to include ec2:CreateVpcEndpoint
- Specify valid subnets (one per AZ) and security groups

SHOULD:
- Create endpoint with appropriate subnets for all needed AZs:
  `aws ec2 create-vpc-endpoint --vpc-id <vpc-id> --service-name com.amazonaws.<region>.<service> --vpc-endpoint-type Interface --subnet-ids <subnet-ids> --security-group-ids <sg-ids>`
- Enable private DNS if needed: add `--private-dns-enabled`

MAY:
- Request quota increase if at endpoint limit
- Contact the endpoint service provider if permission is denied for custom services

## Common Issues

- symptoms: "Duplicate endpoint error"
  diagnosis: "An endpoint for this service already exists in the VPC."
  resolution: "Only one endpoint per VPC per service. Modify the existing endpoint instead."

- symptoms: "Service not found"
  diagnosis: "Service name is incorrect or not available in this region."
  resolution: "Verify service name with describe-vpc-endpoint-services. Check regional availability."

- symptoms: "Authorization error for custom endpoint service"
  diagnosis: "Your account is not in the endpoint service's allowed principals list."
  resolution: "Contact the service provider to add your account to allowed principals."

## Output Format

```yaml
root_cause: "endpoint_creation_failure — <specific_cause>"
evidence:
  - type: service_availability
    content: "<service availability in region>"
  - type: iam_permissions
    content: "<IAM policy evaluation>"
  - type: existing_endpoints
    content: "<existing endpoints for this service>"
severity: HIGH
mitigation:
  immediate: "Fix the specific creation blocker"
  long_term: "Document endpoint creation requirements and automate with IaC"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: LOW — creation failures do not expose data
- Service Disruption Risk: LOW — creation failures do not affect existing endpoints
- Compliance Impact: MEDIUM — endpoint creation affects private connectivity requirements

## Escalation Conditions
- Endpoint creation fails repeatedly with service errors
- Subnet or security group configuration prevents creation
- Service quota limits prevent endpoint creation
- Endpoint creation blocked by VPC endpoint policies
- Multiple endpoint creations fail simultaneously

## Data Sensitivity
- VPC and subnet configurations — INTERNAL
- Security group rules — INTERNAL
- Endpoint policies — INTERNAL
- Service names — INTERNAL
- Never expose VPC architecture details externally

## Prohibited Actions
- Do NOT create endpoints without proper security group configurations
- Do NOT bypass endpoint policy requirements
- Do NOT create endpoints in incorrect subnets
- Do NOT ignore endpoint creation failures
- Do NOT create endpoints without network team coordination
- Do NOT expose endpoint DNS names externally

## Phase 3 — Rollback
- Delete failed endpoint resources
- Clean up orphaned ENIs from failed creation
- Restore security group configurations
- Document creation failure reasons before retrying
- Verify no residual resources remain
- Update endpoint creation procedures

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
