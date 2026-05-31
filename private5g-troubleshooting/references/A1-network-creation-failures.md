---
title: "A1 — Network Creation Failures"
description: "Diagnose AWS Private 5G network creation failures"
status: active
severity: HIGH
triggers: ["cannot create network", "network creation failed", "CreateNetwork error"]
owner: devops-agent
objective: "Resolve Private 5G network creation issues"
context: "Network creation requires valid shipping address, IAM permissions, and service availability in the region."
---

## Phase 1 — Triage

MUST:
- List existing networks: `aws private-networks list-networks`
- Check IAM permissions for private-networks:CreateNetwork
- Verify the region supports AWS Private 5G: `aws private-networks list-networks --region us-east-1`

SHOULD:
- Review CloudTrail for CreateNetwork errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateNetwork --max-results 10`
- Verify shipping address is in a supported US location

MAY:
- Check AWS service quotas: `aws service-quotas get-service-quota --service-code private-networks --quota-code L-12345678`

## Phase 2 — Remediate

MUST:
- Fix IAM permissions if AccessDeniedException
- Provide valid US shipping address for hardware delivery
- Ensure region supports Private 5G service

SHOULD:
- Use CloudFormation or Terraform for repeatable network creation
- Tag networks for cost tracking and organization

MAY:
- Request service quota increase if limits reached

## Safety Ratings

```
safety_ratings:
  - "list-networks: GREEN — read-only enumeration"
  - "CreateNetwork: YELLOW — creates network resource and triggers hardware order"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
  - "service-quotas get-service-quota: GREEN — read-only quota check"
```

## Escalation Conditions

- Network creation fails with internal service errors after IAM and address validation
- Service quota increase request is denied
- Hardware order is stuck in ACKNOWLEDGING status for more than 48 hours
- Region availability issues preventing network creation

## Data Sensitivity

- `aws private-networks list-networks`: LOW — returns network metadata
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history and principal identities
- Network creation parameters: MEDIUM — includes physical shipping address

## Prohibited Actions

- NEVER create a network without confirming the shipping address with the customer
- NEVER delete an existing network to work around creation issues
- NEVER share shipping address information in logs or outputs
- NEVER attempt to create networks in unsupported regions

## Phase 3 — Rollback

1. If a network was created in error, verify no hardware has shipped before deletion
2. Cancel pending orders if possible: check order status via `aws private-networks list-orders --network-arn <arn>`
3. Delete the network only if no hardware has been delivered: `aws private-networks delete-network --network-arn <arn>`
4. Document the failed creation attempt and root cause
5. Notify the operations team if hardware was ordered incorrectly

## Common Issues

- symptoms: "CreateNetwork returns ValidationException for shipping address"
  diagnosis: "Shipping address is invalid or outside supported US locations."
  resolution: "Verify address format and ensure it is in a US location where Private 5G is available."

- symptoms: "CreateNetwork returns AccessDeniedException"
  diagnosis: "IAM principal lacks private-networks:CreateNetwork permission."
  resolution: "Attach policy with private-networks:CreateNetwork and required ec2 permissions."

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
