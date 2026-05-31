---
title: "C1 — Host Resource Group Errors"
description: "Diagnose host resource group creation and management errors"
status: active
severity: HIGH
triggers: ["host resource group error", "cannot create host group", "host allocation failed"]
owner: devops-agent
objective: "Resolve host resource group configuration issues"
context: "Host resource groups manage dedicated host allocation for license tracking. They automate host management for BYOL scenarios."
---

## Phase 1 — Triage

MUST:
- List host resource groups: `aws resource-groups list-groups --filters Name=type,Values=AWS::EC2::HostResourceGroup`
- Check group configuration: `aws resource-groups get-group --group-name <name>`
- Verify license configuration association: `aws license-manager get-license-configuration --license-configuration-arn <arn>`

SHOULD:
- Check dedicated host availability: `aws ec2 describe-hosts --filter "Name=state,Values=available"`
- Verify IAM permissions for host resource group management

MAY:
- Check EC2 host limits: `aws service-quotas get-service-quota --service-code ec2 --quota-code L-949445B0`
- Review host placement group configuration

## Phase 2 — Remediate

MUST:
- Create host resource group with proper license configuration association
- Ensure dedicated hosts have auto-placement enabled: `aws ec2 modify-hosts --host-ids <host-id> --auto-placement on`
- Verify instance type family matches between host and license configuration

SHOULD:
- Allocate dedicated hosts in the required AZs: `aws ec2 allocate-hosts --instance-type <type> --availability-zone <az> --quantity 1 --auto-placement on`
- Configure host recovery for resilience

MAY:
- Set up host maintenance windows
- Implement automated host capacity management

## Safety Ratings

```
safety_ratings:
  - "resource-groups list-groups: GREEN — read-only listing"
  - "ec2 describe-hosts: GREEN — read-only host check"
  - "ec2 allocate-hosts: RED — allocates billable dedicated hosts"
  - "ec2 modify-hosts: YELLOW — changes host configuration"
```

## Escalation Conditions

- Dedicated host allocation fails due to capacity constraints
- Host resource group cannot be associated with license configuration
- Host recovery events cause license tracking discrepancies
- Multiple AZs need host allocation simultaneously

## Data Sensitivity

- `aws ec2 describe-hosts`: MEDIUM — reveals host and instance details
- Host resource group config: LOW — contains group metadata
- License configuration details: MEDIUM — reveals licensing posture
- Host allocation details: MEDIUM — reveals infrastructure capacity

## Prohibited Actions

- NEVER allocate dedicated hosts without cost approval (significant expense)
- NEVER disable auto-placement on hosts in active resource groups
- NEVER release dedicated hosts with running instances
- NEVER mix instance families in a single host resource group

## Phase 3 — Rollback

1. Remove instances from hosts before releasing: `aws ec2 stop-instances --instance-ids <id>`
2. Release unused dedicated hosts: `aws ec2 release-hosts --host-ids <host-id>`
3. Delete host resource group if created in error: `aws resource-groups delete-group --group-name <name>`
4. Verify license tracking is correct after host changes
5. Document host allocation changes for capacity planning

## Common Issues

- symptoms: "Instance launch fails with InsufficientHostCapacity"
  diagnosis: "No available dedicated hosts with capacity for the instance type."
  resolution: "Allocate additional dedicated hosts or terminate unused instances on existing hosts."

## Output Format

```yaml
root_cause: "host_resource_group — <specific_cause>"
evidence:
  - type: host_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix host resource group configuration"
  long_term: "Automate host capacity management"
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
