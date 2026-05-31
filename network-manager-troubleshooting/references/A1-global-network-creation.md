---
title: "A1 — Global Network Creation"
description: "Diagnose global network creation and configuration failures"
status: active
severity: HIGH
triggers: ["cannot create global network", "global network creation failed", "global network state error"]
owner: devops-agent
objective: "Resolve global network creation and initialization issues"
context: "A global network is a logical container for transit gateways, core networks, and other resources. It does not create physical infrastructure."
---

## Phase 1 — Triage

MUST:
- List existing global networks: `aws networkmanager describe-global-networks`
- Verify IAM permissions for `networkmanager:CreateGlobalNetwork`
- Check if a global network already exists (only one is typically needed)

SHOULD:
- Review CloudTrail for CreateGlobalNetwork errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateGlobalNetwork --max-results 10`
- Verify the account is not hitting service quotas

MAY:
- Check AWS Health Dashboard for Network Manager service issues
- Review organization-level SCPs that may block networkmanager actions

## Phase 2 — Remediate

MUST:
- Fix IAM permissions if AccessDeniedException
- Remove conflicting SCPs if applicable
- Request quota increase if limit reached

SHOULD:
- Tag the global network for identification: `aws networkmanager tag-resource --resource-arn <arn> --tags Key=Environment,Value=production`
- Document the global network ID for team reference

MAY:
- Use CloudFormation or Terraform for reproducible deployment
- Set up CloudWatch alarms for global network state changes

## Common Issues

- symptoms: "CreateGlobalNetwork returns AccessDeniedException"
  diagnosis: "IAM policy missing networkmanager:CreateGlobalNetwork permission."
  resolution: "Add networkmanager:* or specific networkmanager:CreateGlobalNetwork to the IAM policy."

- symptoms: "Global network created but no resources visible"
  diagnosis: "Global network is a logical container — resources must be registered separately."
  resolution: "Register transit gateways and create core networks within the global network."

## Safety Ratings

```
safety_ratings:
  - "describe-global-networks: GREEN — read-only enumeration"
  - "CreateGlobalNetwork: YELLOW — creates a logical resource, no infrastructure impact"
  - "tag-resource: GREEN — metadata-only change"
  - "delete-global-network: RED — removes the management container and all associations"
```

## Escalation Conditions

- Global network creation fails with internal service errors after IAM and quota fixes
- Global network stuck in non-AVAILABLE state for more than 15 minutes
- Multiple global networks exist and team is unsure which is authoritative
- Cross-account global network sharing issues requiring RAM configuration

## Data Sensitivity

- `aws networkmanager describe-global-networks`: LOW — returns network metadata only
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history and principal identities
- `aws networkmanager list-tags-for-resource`: LOW — returns resource tags

## Prohibited Actions

- NEVER delete a global network without confirming all registered resources have been deregistered
- NEVER create multiple global networks without explicit architectural justification
- NEVER modify global network configuration in production without change management approval
- NEVER assume a global network creates physical infrastructure

## Phase 3 — Rollback

1. If a global network was created in error, first deregister all transit gateways: `aws networkmanager deregister-transit-gateway --global-network-id <id> --transit-gateway-arn <arn>`
2. Delete any core networks within the global network: `aws networkmanager delete-core-network --core-network-id <id>`
3. Delete the global network: `aws networkmanager delete-global-network --global-network-id <id>`
4. Verify deletion completed: `aws networkmanager describe-global-networks`
5. Document the rollback action and notify the network team

## Output Format

```yaml
root_cause: "global_network_creation — <specific_cause>"
evidence:
  - type: creation_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix creation issue"
  long_term: "Deploy global network via IaC"
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
