---
title: "A1 — Wavelength Zone Issues"
description: "Diagnose Wavelength Zone availability and opt-in problems"
status: active
severity: HIGH
triggers:
  - "wavelength zone"
  - "zone not available"
  - "opt-in wavelength"
  - "enable wavelength"
owner: devops-agent
objective: "Resolve Wavelength Zone opt-in, availability, and configuration issues"
context: "Wavelength Zones must be explicitly opted-in per account. They are separate from standard AZs and Local Zones. Availability depends on carrier partnerships and geographic coverage."
---

## Phase 1 — Triage

MUST:
- List all Wavelength Zones: `aws ec2 describe-availability-zones --filters "Name=zone-type,Values=wavelength-zone" --all-availability-zones`
- Check opt-in status: `aws ec2 describe-availability-zones --filters "Name=zone-type,Values=wavelength-zone" --query 'AvailabilityZones[*].{Zone:ZoneName,State:OptInStatus,Group:GroupName}'`
- Verify account has opted-in to the target zone
- Check for any service quotas: `aws service-quotas list-service-quotas --service-code ec2`

SHOULD:
- Verify the Wavelength Zone is in the desired Region
- Check carrier availability for the target geographic area
- Review account-level restrictions or SCPs

MAY:
- Check AWS documentation for the latest Wavelength Zone availability
- Contact AWS Support for zone-specific issues

## Phase 2 — Remediate

MUST:
- If not opted-in: `aws ec2 modify-availability-zone-group --group-name <zone-group> --opt-in-status opted-in`
- Wait for opt-in to propagate (may take a few minutes)
- Verify opt-in succeeded by re-checking zone status

SHOULD:
- Document which Wavelength Zones are opted-in for the account
- Set up infrastructure-as-code for zone opt-in management

MAY:
- Request access to additional Wavelength Zones through AWS

## Common Issues

- symptoms: "Wavelength Zone not appearing in describe-availability-zones"
  diagnosis: "Zone not opted-in or --all-availability-zones flag not used."
  resolution: "Use --all-availability-zones flag. Opt-in with modify-availability-zone-group."

- symptoms: "Cannot create subnet in Wavelength Zone"
  diagnosis: "Zone not opted-in or VPC CIDR exhausted."
  resolution: "Verify opt-in status. Check VPC CIDR for available address space."

## Output Format

```yaml
root_cause: "wavelength_zone — <specific_cause>"
evidence:
  - type: zone_status
    content: "<opt-in status>"
  - type: zone_availability
    content: "<available Wavelength Zones>"
severity: HIGH
mitigation:
  immediate: "Opt-in to the required Wavelength Zone"
  long_term: "Document and automate zone opt-in management"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Edge deployment configurations reveal application architecture
- Carrier IP assignments expose public-facing endpoints
- Latency data may reveal user geographic distribution
- Classification: INTERNAL - edge infrastructure topology and carrier network details

## Prohibited Actions

- NEVER terminate EC2 instances in Wavelength zones serving live edge traffic
- NEVER modify carrier gateway configurations during peak usage
- NEVER change VPC subnet associations without confirming application connectivity
- NEVER remove carrier IP addresses assigned to production workloads

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

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
