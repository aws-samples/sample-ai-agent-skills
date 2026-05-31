---
title: "A1 — Portal Creation Failures"
description: "Diagnose WorkSpaces Web portal creation failures"
status: active
severity: HIGH
triggers: ["cannot create portal", "portal creation failed", "CreatePortal error"]
owner: devops-agent
objective: "Resolve portal creation issues so the WorkSpaces Web portal becomes ACTIVE"
context: "Portals require network settings, browser settings, and an identity provider. Creation fails if dependencies are missing or misconfigured."
---

## Phase 1 — Triage

MUST:
- Check existing portals: `aws workspaces-web list-portals`
- Verify IAM permissions for workspaces-web:CreatePortal and related actions
- Check if required dependencies exist (network settings, browser settings): `aws workspaces-web list-network-settings` and `aws workspaces-web list-browser-settings`

SHOULD:
- Verify the VPC, subnets, and security groups referenced in network settings exist: `aws ec2 describe-vpcs --vpc-ids <vpc-id>` and `aws ec2 describe-subnets --subnet-ids <subnet-ids>`
- Check CloudTrail for CreatePortal errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreatePortal --max-results 10`

MAY:
- Verify service-linked role exists: `aws iam get-role --role-name AWSServiceRoleForAmazonWorkSpacesWeb`
- Check service quotas for WorkSpaces Web portals

## Phase 2 — Remediate

MUST:
- Create required network settings if missing: `aws workspaces-web create-network-settings --vpc-id <vpc-id> --subnet-ids <subnet-id-1> <subnet-id-2> --security-group-ids <sg-id>`
- Create browser settings if missing: `aws workspaces-web create-browser-settings --browser-policy '{"chromePolicies":{}}'`
- Ensure subnets are in at least two availability zones

SHOULD:
- Verify subnets have routes to the internet (NAT gateway or internet gateway)
- Ensure security groups allow outbound HTTPS (443) traffic

MAY:
- Use CloudFormation or Terraform for repeatable portal deployment
- Create trust store for internal CA certificates

## Safety Ratings

```
safety_ratings:
  - "list-portals: GREEN — read-only enumeration"
  - "list-network-settings: GREEN — read-only enumeration"
  - "create-network-settings: YELLOW — creates VPC association resource"
  - "create-browser-settings: YELLOW — creates browser policy resource"
  - "create-portal: YELLOW — creates portal resource, may incur costs"
```

## Escalation Conditions

- Portal creation fails with internal service error after all dependencies are verified
- Service-linked role creation fails due to organization SCP restrictions
- Portal status stuck in INCOMPLETE for more than 30 minutes
- VPC configuration requires changes that affect other workloads

## Data Sensitivity

- `aws workspaces-web list-portals`: LOW — returns portal metadata
- `aws ec2 describe-vpcs`: LOW — returns VPC metadata
- `aws iam get-role`: LOW — returns role metadata
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history and principal identities

## Prohibited Actions

- NEVER create portals in production accounts without change management approval
- NEVER use default VPC for production portal deployments
- NEVER create network settings with overly permissive security groups (0.0.0.0/0 inbound)
- NEVER skip identity provider configuration — portals without IdP cannot authenticate users

## Phase 3 — Rollback

1. Delete the portal if created in error: `aws workspaces-web delete-portal --portal-arn <portal-arn>`
2. Delete orphaned network settings: `aws workspaces-web delete-network-settings --network-settings-arn <arn>`
3. Delete orphaned browser settings: `aws workspaces-web delete-browser-settings --browser-settings-arn <arn>`
4. Verify no users were given the portal URL before deletion
5. Document the creation attempt and failure reason

## Common Issues

- symptoms: "CreatePortal returns ValidationException about subnets"
  diagnosis: "Subnets must be in at least two different availability zones."
  resolution: "Provide subnet IDs from at least two AZs in the network settings."

## Output Format

```yaml
root_cause: "portal_creation — <specific_cause>"
evidence:
  - type: creation_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix portal creation dependencies"
  long_term: "Use IaC for repeatable portal deployment"
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
