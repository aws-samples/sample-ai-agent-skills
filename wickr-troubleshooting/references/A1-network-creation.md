---
title: "A1 — Network Creation"
description: "Diagnose Wickr network creation failures"
status: active
severity: HIGH
triggers: ["cannot create network", "network creation failed", "CreateNetwork error"]
owner: devops-agent
objective: "Resolve Wickr network creation issues"
context: "Wickr network creation requires IAM permissions, a unique network name, and service availability in the region."
---

## Phase 1 — Triage

MUST:
- List existing networks: `aws wickr list-networks`
- Check IAM permissions for wickr:CreateNetwork
- Review CloudTrail for creation errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateNetwork --max-results 10`

SHOULD:
- Verify the region supports AWS Wickr
- Check if a network with the same name already exists
- Verify service quotas: `aws service-quotas get-service-quota --service-code wickr --quota-code L-XXXXXXXX`

MAY:
- Check AWS Health Dashboard for Wickr service issues
- Review IAM policy for any deny statements affecting Wickr

## Phase 2 — Remediate

MUST:
- Fix IAM permissions if AccessDeniedException
- Use a unique network name if ConflictException
- Ensure region supports Wickr service

SHOULD:
- Use IaC (CloudFormation/Terraform) for repeatable network creation
- Tag networks for organization and cost tracking

MAY:
- Request service quota increase if limits reached
- Set up CloudTrail monitoring for Wickr API calls

## Safety Ratings

```
safety_ratings:
  - "list-networks: GREEN — read-only enumeration"
  - "CreateNetwork: YELLOW — creates new Wickr network resource"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
  - "service-quotas get-service-quota: GREEN — read-only quota check"
```

## Escalation Conditions

- Network creation fails with internal service errors after IAM fixes
- Service quota increase request is denied
- Network creation succeeds but network status remains in CREATING state
- Multiple creation attempts fail across different regions

## Data Sensitivity

- `aws wickr list-networks`: LOW — returns network metadata
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history and principal identities
- Network configuration: LOW — network name and settings

## Prohibited Actions

- NEVER delete an existing network to work around creation issues
- NEVER create networks in unsupported regions
- NEVER share network IDs in public channels
- NEVER create multiple networks without understanding the organizational impact

## Phase 3 — Rollback

1. If a network was created in error, verify no users have been provisioned before deletion
2. Delete the network only if empty: `aws wickr delete-network --network-id <id>`
3. Verify IAM roles and policies created for the network are cleaned up
4. Document the failed creation attempt and root cause
5. Notify the security team if network creation was unauthorized

## Common Issues

- symptoms: "CreateNetwork returns AccessDeniedException"
  diagnosis: "IAM principal lacks wickr:CreateNetwork permission."
  resolution: "Attach policy with wickr:CreateNetwork and required IAM permissions."

- symptoms: "CreateNetwork returns ConflictException"
  diagnosis: "A network with the same name already exists."
  resolution: "Use a unique network name or manage the existing network."

## Output Format

```yaml
root_cause: "network_creation — <specific_cause>"
evidence:
  - type: creation_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix creation blocker"
  long_term: "Automate network creation via IaC"
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
