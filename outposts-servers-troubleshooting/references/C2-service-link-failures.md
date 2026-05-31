---
title: "C2 — Service Link Failures"
description: "Diagnose service link connectivity failures between Outposts Server and parent Region"
status: active
severity: CRITICAL
triggers: ["service link down", "outpost disconnected", "cannot reach region", "ConnectedStatus 0", "control plane unavailable"]
owner: devops-agent
objective: "Restore service link connectivity between the Outposts Server and the parent AWS Region"
context: "The service link is the network connection from the Outposts Server to the parent Region. It carries control plane traffic, CloudWatch metrics, and EBS snapshots. Running instances continue during brief outages."
---

## Phase 1 — Triage

MUST:
- Check ConnectedStatus metric: `aws cloudwatch get-metric-statistics --namespace AWS/Outposts --metric-name ConnectedStatus --dimensions Name=OutpostId,Value=<outpost-id> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Verify Outpost lifecycle status: `aws outposts get-outpost --outpost-id <outpost-id> --query '{Status:LifeCycleStatus,Name:Name}'`
- Check for recent service events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=outposts.amazonaws.com --max-results 20`

SHOULD:
- Verify network path from edge site to Region (firewall rules, proxy settings)
- Check DNS resolution for AWS service endpoints from the edge site
- Verify NTP synchronization (TLS handshake failures can result from clock skew)

MAY:
- Check AWS Health Dashboard for regional Outposts service issues
- Review edge site network equipment logs for link failures

## Phase 2 — Remediate

MUST:
- Ensure TCP 443 outbound is allowed from the Outpost to Region endpoints
- Verify DNS resolution for outposts.<region>.amazonaws.com and related endpoints
- Confirm physical network path is operational (cables, switches, routers)

SHOULD:
- Check for MTU issues on the network path (service link requires 1500 byte MTU minimum)
- Verify no proxy or deep packet inspection is interfering with the service link
- Test with traceroute/mtr from the edge site to Region endpoints

MAY:
- Implement redundant network paths for the service link
- Configure network monitoring for early detection of link degradation

## Common Issues

- symptoms: "ConnectedStatus metric shows 0 for extended period"
  diagnosis: "Service link network path is blocked or degraded."
  resolution: "Check firewall rules for TCP 443 outbound. Verify DNS resolution. Check physical network path. Ensure no proxy interference."

## Safety Ratings

```
safety_ratings:
  - "cloudwatch get-metric-statistics: GREEN — read-only metric check"
  - "get-outpost: GREEN — read-only status check"
  - "Firewall rule changes: RED — modifies network security at edge site"
  - "Network equipment changes: RED — can affect all edge site connectivity"
```

## Escalation Conditions

- Service link has been down for more than 1 hour with no identified network cause
- Running instances are affected (should not happen with brief outages)
- Service link restoration requires changes to corporate firewall policies
- Physical network infrastructure damage suspected

## Data Sensitivity

- `aws cloudwatch get-metric-statistics`: LOW — returns connectivity metrics
- Network path diagnostics: MEDIUM — reveals network topology between edge and Region
- Firewall configurations: HIGH — reveals security posture of edge site
- Edge site physical network details: HIGH — reveals infrastructure layout

## Prohibited Actions

- NEVER disable firewalls entirely to restore service link connectivity
- NEVER modify corporate network infrastructure without change management approval
- NEVER assume running instances are unaffected during extended disconnection
- NEVER attempt to manually configure the service link endpoint on the server

## Phase 3 — Rollback

1. If firewall rules were modified, revert to the documented previous state
2. If DNS settings were changed at the edge site, restore original configuration
3. If network equipment was reconfigured, revert changes and verify other services are unaffected
4. Verify service link reconnects after rollback (ConnectedStatus returns to 1)
5. Document the root cause and network path for future reference

## Output Format

```yaml
root_cause: "service_link_failure — <specific_cause>"
evidence:
  - type: connectivity_status
    content: "<ConnectedStatus metrics and network diagnostics>"
severity: CRITICAL
mitigation:
  immediate: "Restore service link connectivity"
  long_term: "Implement redundant network paths and monitoring"
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
