---
title: "F1 — Automated Discovery"
description: "Diagnose automated software discovery issues"
status: active
severity: MEDIUM
triggers: ["automated discovery", "software discovery", "ssm inventory", "license discovery"]
owner: devops-agent
objective: "Resolve automated software discovery issues"
context: "Automated discovery uses SSM inventory to find software installations on managed instances. Requires SSM Agent and proper IAM permissions."
---

## Phase 1 — Triage

MUST:
- Check discovery status: `aws license-manager list-resource-inventory --filters '[{"Name":"Platform","Values":["LINUX","WINDOWS"],"Condition":"BEGINS_WITH"}]'`
- Verify SSM managed instances: `aws ssm describe-instance-information --query 'InstanceInformationList[*].{Id:InstanceId,PingStatus:PingStatus,Platform:PlatformName}'`
- Check SSM inventory collection: `aws ssm list-inventory-entries --instance-id <id> --type-name AWS:Application`

SHOULD:
- Verify SSM inventory association exists: `aws ssm list-associations --association-filter-list Key=AssociationName,Value=AWS-GatherSoftwareInventory`
- Check IAM instance profile for SSM permissions

MAY:
- Review discovery rules in License Manager
- Check for instances not reporting inventory

## Phase 2 — Remediate

MUST:
- Install SSM Agent on unmanaged instances
- Attach AmazonSSMManagedInstanceCore policy to instance IAM roles
- Create inventory association if missing: `aws ssm create-association --name AWS-GatherSoftwareInventory --targets Key=InstanceIds,Values=<id>`

SHOULD:
- Configure inventory collection schedule (default is 30 minutes)
- Set up discovery rules to map software to license configurations

MAY:
- Use SSM hybrid activation for on-premises servers: `aws ssm create-activation --iam-role <role-name> --registration-limit 10`
- Set up CloudWatch dashboards for discovery metrics

## Safety Ratings

```
safety_ratings:
  - "license-manager list-resource-inventory: GREEN — read-only listing"
  - "ssm describe-instance-information: GREEN — read-only instance check"
  - "ssm create-association: YELLOW — creates inventory collection on instances"
  - "ssm create-activation: YELLOW — enables hybrid management"
```

## Escalation Conditions

- SSM Agent cannot be installed on critical servers
- Inventory collection fails on a large number of instances
- Discovery rules produce incorrect software-to-license mappings
- On-premises servers cannot connect to SSM endpoints

## Data Sensitivity

- `aws ssm describe-instance-information`: MEDIUM — reveals instance details
- Software inventory: MEDIUM — reveals installed applications
- SSM hybrid activations: HIGH — enables remote management of on-premises servers
- Discovery rules: LOW — contains mapping configuration

## Prohibited Actions

- NEVER install SSM Agent on instances without proper authorization
- NEVER create hybrid activations with unlimited registration limits
- NEVER ignore discovery discrepancies — they indicate compliance gaps
- NEVER share software inventory data outside the compliance team

## Phase 3 — Rollback

1. Delete inventory associations if created incorrectly: `aws ssm delete-association --association-id <id>`
2. Deregister hybrid instances if activated in error: `aws ssm deregister-managed-instance --instance-id <id>`
3. Remove discovery rules if they produce incorrect mappings
4. Verify license tracking is correct after discovery changes
5. Document discovery configuration changes

## Common Issues

- symptoms: "Resource inventory shows no software for managed instances"
  diagnosis: "SSM inventory association not created or inventory collection not running."
  resolution: "Create AWS-GatherSoftwareInventory association. Verify SSM Agent is running and has network access."

## Output Format

```yaml
root_cause: "automated_discovery — <specific_cause>"
evidence:
  - type: discovery_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix discovery configuration"
  long_term: "Automate SSM Agent deployment and inventory collection"
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
