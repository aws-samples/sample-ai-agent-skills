---
title: "C2 — Dedicated Host Management"
description: "Diagnose dedicated host lifecycle and management issues"
status: active
severity: HIGH
triggers: ["dedicated host error", "host management", "host capacity", "BYOL host"]
owner: devops-agent
objective: "Resolve dedicated host management issues for license compliance"
context: "Dedicated hosts provide physical server isolation for BYOL licensing. They require careful capacity management and license tracking."
---

## Phase 1 — Triage

MUST:
- List dedicated hosts: `aws ec2 describe-hosts --query 'Hosts[*].{Id:HostId,State:State,Type:InstanceType,AZ:AvailabilityZone,AutoPlacement:AutoPlacement}'`
- Check host capacity: `aws ec2 describe-hosts --host-ids <host-id> --query 'Hosts[*].AvailableCapacity'`
- Verify license configuration tracking: `aws license-manager list-usage-for-license-configuration --license-configuration-arn <arn>`

SHOULD:
- Check host recovery status: `aws ec2 describe-hosts --host-ids <host-id> --query 'Hosts[*].HostRecovery'`
- Review instances on the host: `aws ec2 describe-instances --filters "Name=host-id,Values=<host-id>"`

MAY:
- Check host maintenance events: `aws ec2 describe-host-reservation-offerings`
- Review host allocation history in CloudTrail

## Phase 2 — Remediate

MUST:
- Ensure hosts are in available state for instance placement
- Enable auto-placement for hosts in resource groups: `aws ec2 modify-hosts --host-ids <host-id> --auto-placement on`
- Verify instance type family compatibility

SHOULD:
- Enable host recovery: `aws ec2 modify-hosts --host-ids <host-id> --host-recovery on`
- Balance instances across multiple hosts for resilience

MAY:
- Purchase host reservations for cost savings
- Implement automated host scaling based on demand

## Safety Ratings

```
safety_ratings:
  - "ec2 describe-hosts: GREEN — read-only host listing"
  - "ec2 modify-hosts: YELLOW — changes host configuration"
  - "ec2 allocate-hosts: RED — allocates billable dedicated hosts"
  - "ec2 release-hosts: RED — releases hosts, may affect running instances"
```

## Escalation Conditions

- Host enters under-assessment state due to hardware issues
- Host recovery fails and instances are lost
- Capacity constraints prevent host allocation in required AZ
- License compliance requires immediate host changes

## Data Sensitivity

- `aws ec2 describe-hosts`: MEDIUM — reveals host and capacity details
- Instance placement: MEDIUM — reveals workload distribution
- Host reservations: MEDIUM — reveals financial commitments
- License usage: MEDIUM — reveals compliance posture

## Prohibited Actions

- NEVER release hosts with running production instances
- NEVER disable host recovery on production hosts
- NEVER allocate hosts without understanding the cost implications
- NEVER move instances between hosts without verifying license compliance

## Phase 3 — Rollback

1. If host configuration was changed incorrectly, revert: `aws ec2 modify-hosts --host-ids <host-id> --auto-placement <original-value>`
2. If hosts were allocated unnecessarily, release after migrating instances
3. Verify all instances are properly tracked after host changes
4. Update license configuration counts if host changes affect capacity
5. Document host management changes

## Common Issues

- symptoms: "Host shows state 'under-assessment'"
  diagnosis: "AWS detected potential hardware issue on the dedicated host."
  resolution: "Wait for assessment to complete. If host recovery is enabled, instances will be migrated automatically."

## Output Format

```yaml
root_cause: "dedicated_host — <specific_cause>"
evidence:
  - type: host_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix dedicated host issue"
  long_term: "Implement host recovery and capacity planning"
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
