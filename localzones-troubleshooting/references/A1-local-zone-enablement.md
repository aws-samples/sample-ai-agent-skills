---
title: "A1 — Local Zone Enablement"
description: "Diagnose Local Zone opt-in and enablement failures"
status: active
severity: HIGH
triggers: ["cannot enable Local Zone", "opt-in failed", "Local Zone not visible", "zone not available"]
owner: devops-agent
objective: "Resolve Local Zone enablement issues"
context: "Local Zones require explicit opt-in per zone group. Without opt-in, zones are not visible for subnet creation or instance launches."
---

## Phase 1 — Triage

MUST:
- Check current zone status: `aws ec2 describe-availability-zones --all-availability-zones --filters "Name=zone-type,Values=local-zone" --query 'AvailabilityZones[*].{ZoneName:ZoneName,State:State,OptInStatus:OptInStatus,GroupName:GroupName}'`
- Verify IAM permissions for ec2:ModifyAvailabilityZoneGroup
- Check if the desired Local Zone exists in the region

SHOULD:
- Verify the account is not restricted from Local Zone access
- Check if the zone group name is correct

MAY:
- Review CloudTrail for ModifyAvailabilityZoneGroup errors
- Check AWS Health Dashboard for Local Zone availability

## Phase 2 — Remediate

MUST:
- Opt in to the Local Zone group: `aws ec2 modify-availability-zone-group --group-name <group-name> --opt-in-status opted-in`
- Wait for opt-in to complete (may take several minutes)
- Verify zone state changes to available: `aws ec2 describe-availability-zones --zone-names <zone-name>`

SHOULD:
- Document which Local Zones are enabled for the account
- Enable Local Zones in all accounts that need access (opt-in is per-account)

MAY:
- Use AWS Organizations SCPs to control which accounts can enable Local Zones
- Automate opt-in across accounts using CloudFormation or scripts

## Safety Ratings

```
safety_ratings:
  - "describe-availability-zones: GREEN — read-only zone enumeration"
  - "modify-availability-zone-group opt-in: YELLOW — enables zone access, may incur costs when resources are created"
  - "modify-availability-zone-group opt-out: RED — prevents new resource creation, may orphan existing resources"
  - "CloudTrail lookup: GREEN — read-only audit log"
```

## Escalation Conditions

- Opt-in fails with internal service error after multiple attempts
- Account appears to be restricted from Local Zone access
- SCP prevents opt-in and requires organization admin intervention
- Local Zone shows as available but subnets cannot be created

## Data Sensitivity

- `aws ec2 describe-availability-zones`: LOW — zone metadata only
- `aws ec2 modify-availability-zone-group`: LOW — zone enablement action
- CloudTrail events: MEDIUM — reveals IAM principals and actions
- Account zone configuration: LOW — opt-in status information

## Prohibited Actions

- NEVER opt out of a Local Zone with active resources without migrating them first
- NEVER enable Local Zones in production accounts without cost impact analysis
- NEVER assume opt-in is instant — allow several minutes for propagation
- NEVER enable all available Local Zones without a specific business need

## Phase 3 — Rollback

1. If Local Zone was enabled in error, first verify no resources exist in the zone
2. Opt out: `aws ec2 modify-availability-zone-group --group-name <group-name> --opt-in-status not-opted-in`
3. Verify no subnets, instances, or volumes remain in the zone before opt-out
4. Document the opt-out action and update infrastructure documentation

## Common Issues

- symptoms: "Local Zone does not appear in describe-availability-zones"
  diagnosis: "Zone is not opted in. Default describe only shows opted-in zones."
  resolution: "Use --all-availability-zones flag to see all zones. Opt in with modify-availability-zone-group."

- symptoms: "ModifyAvailabilityZoneGroup returns UnauthorizedOperation"
  diagnosis: "IAM policy missing ec2:ModifyAvailabilityZoneGroup permission or SCP blocking."
  resolution: "Add ec2:ModifyAvailabilityZoneGroup to IAM policy. Check SCPs for deny rules."

## Output Format

```yaml
root_cause: "local_zone_enablement — <specific_cause>"
evidence:
  - type: zone_status
    content: "<zone opt-in status and error details>"
severity: HIGH
mitigation:
  immediate: "Enable Local Zone"
  long_term: "Automate zone enablement in account provisioning"
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
