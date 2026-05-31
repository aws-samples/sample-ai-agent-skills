---
title: "F1 — Rack vs Server Differences"
description: "Clarify differences between Outposts racks and Outposts Servers"
status: active
severity: MEDIUM
triggers: ["rack vs server", "outpost form factor", "which outpost type", "server limitations vs rack"]
owner: devops-agent
objective: "Help users understand the differences between Outposts racks (42U) and Outposts Servers (1U/2U) to avoid misconfiguration"
context: "Outposts racks and Outposts Servers are different form factors with different capabilities, supported services, and use cases. Confusion between them causes incorrect architecture decisions."
---

## Phase 1 — Triage

MUST:
- Identify which form factor is deployed: `aws outposts get-outpost --outpost-id <outpost-id> --query '{Name:Name,Arn:OutpostArn,LifeCycleStatus:LifeCycleStatus}'`
- List supported instance types: `aws outposts get-outpost-instance-types --outpost-id <outpost-id>`
- Check Outpost assets to determine form factor: `aws outposts list-assets --outpost-identifier <outpost-id> --query 'Assets[*].{AssetId:AssetId,AssetType:AssetType}'`

SHOULD:
- Verify which AWS services are expected vs actually available
- Check if the user's architecture requires rack-only features (S3, RDS, EKS)
- Review instance family availability

MAY:
- Compare pricing models between rack and server deployments
- Evaluate if the workload is better suited for a different form factor

## Phase 2 — Remediate

MUST:
- Confirm the deployed form factor matches the workload requirements
- Adjust architecture to use only services supported by the deployed form factor
- For servers: use only EC2 and EBS; do not attempt S3, RDS, EKS, or ECS on Outposts

SHOULD:
- Document the capabilities and limitations of the deployed form factor
- Update architecture diagrams to reflect actual Outpost capabilities

MAY:
- Plan migration to rack form factor if additional services are required
- Implement hybrid patterns using Region services for unsupported features

## Common Issues

- symptoms: "Attempting to create S3 bucket on Outposts Server returns error"
  diagnosis: "S3 on Outposts is only available on rack form factor, not servers."
  resolution: "Use Region S3 via service link, or migrate to Outposts rack if local S3 is required."

## Safety Ratings

```
safety_ratings:
  - "get-outpost: GREEN — read-only Outpost details"
  - "get-outpost-instance-types: GREEN — read-only capacity check"
  - "list-assets: GREEN — read-only asset enumeration"
  - "Architecture changes: YELLOW — may require workload migration"
```

## Escalation Conditions

- Workload requires services not available on the deployed form factor
- Migration from server to rack form factor is needed
- Architecture decisions were made based on incorrect form factor assumptions
- Cost implications of form factor change require management approval

## Data Sensitivity

- `aws outposts get-outpost`: LOW — returns Outpost metadata
- `aws outposts list-assets`: LOW — returns hardware asset information
- Architecture documentation: MEDIUM — reveals infrastructure design decisions
- Pricing comparisons: LOW — publicly available information

## Prohibited Actions

- NEVER assume rack capabilities are available on server form factors
- NEVER attempt to deploy rack-only services (S3, RDS, EKS, ECS) on Outposts Servers
- NEVER recommend form factor changes without cost and timeline analysis
- NEVER conflate rack and server documentation when providing guidance

## Phase 3 — Rollback

1. If services were incorrectly configured for the wrong form factor, remove the configurations
2. If architecture decisions need revision, document the correct capabilities
3. If workloads were deployed expecting unavailable services, plan migration to appropriate platform
4. Update all documentation to reflect the correct form factor capabilities
5. Communicate form factor limitations to all stakeholders

## Output Format

```yaml
root_cause: "form_factor_confusion — <specific_cause>"
evidence:
  - type: form_factor
    content: "<deployed form factor and capability mismatch>"
severity: MEDIUM
mitigation:
  immediate: "Clarify form factor capabilities"
  long_term: "Document and communicate form factor differences"
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
