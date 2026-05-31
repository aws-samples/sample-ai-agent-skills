---
title: "A2 — Server Activation Failures"
description: "Diagnose Outposts Server activation and provisioning failures"
status: active
severity: CRITICAL
triggers: ["server not activating", "activation failed", "outpost stuck provisioning", "lifecycle status not active"]
owner: devops-agent
objective: "Resolve Outposts Server activation failures after physical installation"
context: "Activation requires physical installation, network connectivity (service link), DNS resolution, and proper AWS account configuration. The server must reach ACTIVE lifecycle status."
---

## Phase 1 — Triage

MUST:
- Check Outpost lifecycle status: `aws outposts get-outpost --outpost-id <outpost-id> --query '{Status:LifeCycleStatus,Name:Name,Id:OutpostId}'`
- Verify service link connectivity via CloudWatch: `aws cloudwatch get-metric-statistics --namespace AWS/Outposts --metric-name ConnectedStatus --dimensions Name=OutpostId,Value=<outpost-id> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Review CloudTrail for activation events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=outposts.amazonaws.com --max-results 20`

SHOULD:
- Verify physical installation: power, network cabling, and environmental conditions
- Check firewall rules allow service link traffic to the parent Region
- Verify DNS resolution for AWS service endpoints from the edge site

MAY:
- Check AWS Health Dashboard for Outposts service issues
- Review VPC and subnet configuration for the Outpost

## Phase 2 — Remediate

MUST:
- Ensure service link network path is open (TCP 443 to Region endpoints)
- Verify DNS resolution works for outposts.<region>.amazonaws.com
- Confirm physical installation meets AWS specifications (power, cooling, cabling)

SHOULD:
- Test network connectivity from the edge site to the parent Region
- Verify NTP synchronization at the edge site

MAY:
- Engage AWS Support for on-site activation assistance
- Review network topology for proxy or firewall interference

## Common Issues

- symptoms: "Outpost lifecycle status stuck at PENDING"
  diagnosis: "Service link cannot reach the parent Region."
  resolution: "Verify firewall allows TCP 443 outbound to Region endpoints. Check DNS resolution. Verify physical network cabling."

## Safety Ratings

```
safety_ratings:
  - "get-outpost: GREEN — read-only status check"
  - "cloudwatch get-metric-statistics: GREEN — read-only metric retrieval"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
  - "Firewall rule changes: RED — modifies network security posture at edge site"
```

## Escalation Conditions

- Server has been in non-ACTIVE status for more than 48 hours after physical installation
- Service link connectivity cannot be established despite correct network configuration
- Hardware indicators show errors (LEDs, physical damage during shipping)
- Activation requires on-site AWS technician visit

## Data Sensitivity

- `aws outposts get-outpost`: LOW — returns Outpost metadata and status
- `aws cloudwatch get-metric-statistics`: LOW — returns connectivity metrics
- Network configuration details: MEDIUM — reveals edge site network topology
- Physical site access information: HIGH — reveals physical security details

## Prohibited Actions

- NEVER attempt to power cycle or physically reset the Outposts Server without AWS Support guidance
- NEVER modify service link network configuration without documenting the current state
- NEVER bypass firewall rules permanently to troubleshoot activation
- NEVER open the server chassis or attempt hardware-level troubleshooting

## Phase 3 — Rollback

1. If firewall rules were modified for troubleshooting, revert to the original configuration
2. If DNS settings were changed, restore the original DNS configuration
3. If VPC/subnet associations were modified, revert to the pre-troubleshooting state
4. Document all network changes made during activation troubleshooting
5. If activation cannot be completed, engage AWS Support with full diagnostic details

## Output Format

```yaml
root_cause: "activation_failure — <specific_cause>"
evidence:
  - type: lifecycle_status
    content: "<outpost status and connectivity details>"
severity: CRITICAL
mitigation:
  immediate: "Resolve activation blocker"
  long_term: "Document activation prerequisites and network requirements"
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
