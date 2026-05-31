---
title: "B1 — Attachment Failures"
description: "Diagnose VPC, VPN, and Connect attachment creation and state failures"
status: active
severity: HIGH
triggers: ["attachment failed", "attachment rejected", "attachment stuck", "cannot create attachment", "VPC attachment error"]
owner: devops-agent
objective: "Resolve attachment creation, acceptance, and lifecycle issues"
context: "Attachments connect VPCs, VPNs, and Connect peers to the core network. Attachments must match the core network policy's attachment-policies to be accepted."
---

## Phase 1 — Triage

MUST:
- List all attachments: `aws networkmanager list-attachments --core-network-id <id>`
- Get specific attachment details: `aws networkmanager get-vpc-attachment --attachment-id <id>`
- Check core network policy attachment-policies: `aws networkmanager get-core-network-policy --core-network-id <id>`

SHOULD:
- Verify the VPC/resource exists and is in the correct region
- Check if the attachment tag matches any attachment-policy condition
- Review CloudTrail for CreateVpcAttachment or CreateSiteToSiteVpnAttachment errors

MAY:
- Check VPC subnet availability in the specified AZs
- Verify cross-account RAM sharing if attachment is cross-account

## Phase 2 — Remediate

MUST:
- For REJECTED attachments: update tags to match an attachment-policy condition, then recreate
- For FAILED attachments: delete and recreate after fixing the root cause
- Ensure the VPC has subnets in the core network edge location's AZs

SHOULD:
- Verify attachment-policy conditions in the core network policy match the resource tags
- Accept pending attachments if manual acceptance is configured: `aws networkmanager accept-attachment --attachment-id <id>`

MAY:
- Update the core network policy to add new attachment-policy rules
- Use CloudFormation for attachment lifecycle management

## Common Issues

- symptoms: "Attachment stuck in REJECTED state"
  diagnosis: "Resource tags do not match any attachment-policy condition in the core network policy."
  resolution: "Update resource tags to match an attachment-policy condition, delete the rejected attachment, and recreate."

- symptoms: "VPC attachment fails with subnet error"
  diagnosis: "VPC does not have subnets in the required AZs for the core network edge location."
  resolution: "Create subnets in the required AZs or specify different subnet IDs."

## Safety Ratings

```
safety_ratings:
  - "list-attachments: GREEN — read-only enumeration"
  - "get-vpc-attachment: GREEN — read-only detail retrieval"
  - "create-vpc-attachment: YELLOW — creates network attachment, affects routing"
  - "delete-attachment: RED — removes network connectivity for attached VPC"
  - "accept-attachment: YELLOW — enables network connectivity"
```

## Escalation Conditions

- Attachment creation fails with internal errors after fixing all configuration issues
- Production VPC attachment enters FAILED state causing connectivity loss
- Cross-account attachment requires RAM sharing changes affecting multiple teams
- Multiple attachments failing simultaneously indicating a core network issue

## Data Sensitivity

- `aws networkmanager list-attachments`: LOW — returns attachment metadata
- `aws networkmanager get-vpc-attachment`: MEDIUM — reveals VPC and subnet configuration
- `aws networkmanager get-core-network-policy`: MEDIUM — reveals network segmentation strategy

## Prohibited Actions

- NEVER delete a production attachment without confirming failover connectivity exists
- NEVER accept attachments from unknown or unverified accounts
- NEVER modify attachment tags in production without understanding the attachment-policy implications
- NEVER force-delete an attachment that is in DELETING state — wait for completion

## Phase 3 — Rollback

1. If an attachment was deleted in error, recreate it: `aws networkmanager create-vpc-attachment --core-network-id <id> --vpc-arn <arn> --subnet-arns <arns> --tags Key=segment,Value=<segment>`
2. If an attachment was accepted in error, delete it: `aws networkmanager delete-attachment --attachment-id <id>`
3. Verify routing is restored by checking segment routes: `aws networkmanager get-network-routes --global-network-id <gn-id> --route-table-identifier Type=CORE_NETWORK_SEGMENT,CoreNetworkId=<cn-id>,SegmentName=<segment>`
4. Confirm VPC connectivity from affected workloads
5. Document the rollback and notify affected teams

## Output Format

```yaml
root_cause: "attachment_failure — <specific_cause>"
evidence:
  - type: attachment_state
    content: "<attachment details and error>"
severity: HIGH
mitigation:
  immediate: "Fix attachment configuration and recreate"
  long_term: "Implement attachment-policy validation in CI/CD"
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
