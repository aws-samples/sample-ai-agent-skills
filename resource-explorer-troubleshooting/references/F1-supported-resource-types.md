---
title: "F1 — Supported Resource Types"
description: "Diagnose issues related to supported and unsupported resource types"
status: active
severity: LOW
triggers: ["resource type not supported", "unsupported resource", "resource not indexed", "which resources are supported"]
owner: devops-agent
objective: "Identify supported vs unsupported resource types and provide alternatives"
context: "Resource Explorer supports a subset of AWS resource types. Unsupported types never appear in search results."
---

## Phase 1 — Triage

MUST:
- List supported resource types: `aws resource-explorer-2 list-supported-resource-types --query 'ResourceTypes[*].{Service:Service,ResourceType:ResourceType}'`
- Check if the specific resource type is supported: `aws resource-explorer-2 list-supported-resource-types --query "ResourceTypes[?ResourceType=='<type>']"`
- Verify the resource exists using the service-specific API

SHOULD:
- Check if the resource type was recently added to supported list
- Review AWS documentation for the latest supported types

MAY:
- Compare supported types against the resources in the account
- Identify coverage gaps for compliance requirements

## Phase 2 — Remediate

MUST:
- For unsupported types, use alternative discovery methods:
  - AWS Config: `aws configservice select-resource-config --expression "SELECT * WHERE resourceType = '<type>'"`
  - Service-specific APIs: `aws <service> describe-<resources>`
  - Resource Groups Tagging API: `aws resourcegroupstaggingapi get-resources --resource-type-filters <type>`

SHOULD:
- Document which resource types are not covered by Resource Explorer
- Set up AWS Config rules for unsupported resource types

MAY:
- Request resource type support through AWS support or feature requests
- Build custom inventory scripts for unsupported types

## Safety Ratings

```
safety_ratings:
  - "resource-explorer-2 list-supported-resource-types: GREEN — read-only type listing"
  - "configservice select-resource-config: GREEN — read-only Config query"
  - "resourcegroupstaggingapi get-resources: GREEN — read-only tag query"
  - "service-specific describe: GREEN — read-only resource listing"
```

## Escalation Conditions

- Critical resource types for compliance are not supported
- Supported resource types are not being indexed despite ACTIVE index
- Resource type support changes affect existing automation
- Coverage gaps create security blind spots

## Data Sensitivity

- `aws resource-explorer-2 list-supported-resource-types`: LOW — public information
- `aws configservice select-resource-config`: MEDIUM — reveals resource configurations
- `aws resourcegroupstaggingapi get-resources`: MEDIUM — reveals resource tags

## Prohibited Actions

- NEVER assume all AWS resource types are supported by Resource Explorer
- NEVER rely solely on Resource Explorer for compliance inventory
- NEVER dismiss unsupported resource types as unimportant
- NEVER build security monitoring exclusively on Resource Explorer

## Phase 3 — Rollback

1. No rollback needed for read-only operations
2. If alternative discovery methods were configured, verify they are working
3. Document the supported vs unsupported resource type mapping
4. Update team documentation with alternative discovery methods
5. Review coverage gaps periodically as AWS adds new supported types

## Common Issues

- symptoms: "AWS::RDS::DBCluster resources don't appear in search results"
  diagnosis: "The resource type may not be supported by Resource Explorer."
  resolution: "Check list-supported-resource-types. If not supported, use AWS Config or RDS describe-db-clusters for discovery."

## Output Format

```yaml
root_cause: "supported_resource_types — <specific_cause>"
evidence:
  - type: type_check
    content: "<supported types details>"
severity: LOW
mitigation:
  immediate: "Use alternative discovery for unsupported types"
  long_term: "Implement comprehensive inventory with AWS Config"
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
