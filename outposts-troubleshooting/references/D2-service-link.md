---
title: "D2 — Service Link Issues"
description: "Diagnose service link connectivity between Outpost and parent AWS Region"
status: active
severity: CRITICAL
triggers:
  - "service link"
  - "region connection"
  - "outpost API calls failing"
  - "cannot manage outpost"
owner: devops-agent
objective: "Restore and maintain service link connectivity between the Outpost and the parent Region"
context: "The service link is the encrypted connection between the Outpost and the parent AWS Region. It carries management traffic, API calls, and metrics. Without it, new operations fail but existing workloads continue."
---

## Phase 1 — Triage

MUST:
- Check connectivity metric: `aws cloudwatch get-metric-statistics --namespace AWS/Outposts --metric-name ConnectedStatus --dimensions Name=OutpostId,Value=<outpost-id> --start-time <start> --end-time <end> --period 60 --statistics Average`
- Verify Outpost status: `aws outposts get-outpost --outpost-id <outpost-id>`
- Check for AWS Health events: `aws health describe-events --filter "services=OUTPOSTS"`
- Review CloudTrail for recent changes: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=outposts.amazonaws.com --max-results 10`

SHOULD:
- Verify network path between Outpost and AWS Region endpoints
- Check on-premises firewall rules (UDP 443 must be allowed outbound)
- Verify DNS resolution for AWS service endpoints from the Outpost network
- Check WAN link utilization and quality

MAY:
- Review network device logs for the service link path
- Check for MTU issues (minimum 1500 bytes required)

## Phase 2 — Remediate

MUST:
- If firewall blocking: allow UDP 443 outbound to AWS service endpoints
- If DNS failure: configure DNS resolution for AWS endpoints
- If WAN link down: restore the network path to the AWS Region

SHOULD:
- Implement redundant WAN links for service link resilience
- Set up CloudWatch alarms on ConnectedStatus metric
- Document the service link network path for troubleshooting

MAY:
- Work with AWS to review service link configuration
- Implement network monitoring for the WAN path

## Common Issues

- symptoms: "ConnectedStatus metric shows 0"
  diagnosis: "Service link is completely down — network path or firewall issue."
  resolution: "Check physical network, firewall rules (UDP 443), and DNS resolution."

- symptoms: "Intermittent API failures from Outpost"
  diagnosis: "Service link experiencing packet loss or high latency."
  resolution: "Check WAN link quality, bandwidth utilization, and network device health."

## Output Format

```yaml
root_cause: "service_link — <specific_cause>"
evidence:
  - type: connected_status
    content: "<connectivity metric>"
  - type: network_path
    content: "<network diagnostics>"
severity: CRITICAL
mitigation:
  immediate: "Restore service link connectivity"
  long_term: "Implement redundant WAN links and connectivity monitoring"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: LOW — service link is an encrypted management channel
- Service Disruption Risk: HIGH — service link failure affects Outpost management
- Compliance Impact: HIGH — service link is required for Outpost operations

## Escalation Conditions
- Service link goes down completely
- Service link latency exceeds acceptable thresholds
- Service link bandwidth insufficient for management traffic
- Service link failover does not work as expected
- Service link encryption issues detected

## Data Sensitivity
- Service link configuration — INTERNAL
- Bandwidth and latency metrics — INTERNAL
- Failover configurations — INTERNAL
- Encryption settings — INTERNAL
- Never expose service link connection details

## Prohibited Actions
- Do NOT modify service link configurations without AWS Support guidance
- Do NOT bypass service link encryption
- Do NOT ignore service link health warnings
- Do NOT route non-management traffic through the service link
- Do NOT change service link network path without coordination
- Do NOT disable service link monitoring

## Phase 3 — Rollback
- Restore previous service link configurations
- Re-establish service link connectivity
- Verify Outpost management operations after recovery
- Restore monitoring configurations
- Document service link issues and resolution steps
- Update network documentation with service link requirements

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
