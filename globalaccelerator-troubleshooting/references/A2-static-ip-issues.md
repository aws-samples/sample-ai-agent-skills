---
title: "A2 — Global Accelerator Static IP Issues"
description: "Diagnose issues with Global Accelerator static anycast IP addresses"
status: active
severity: HIGH
triggers:
  - "static IP not working"
  - "anycast IP unreachable"
  - "IP address issues"
  - "BYOIP problems"
owner: devops-agent
objective: "Resolve issues with Global Accelerator static anycast IPs including reachability and BYOIP"
context: "Each accelerator gets 2 static anycast IPs that are immutable. BYOIP allows using your own IP ranges. IPs are advertised from AWS edge locations globally."
---

## Phase 1 — Triage

MUST:
- Get accelerator details including IPs: `aws globalaccelerator describe-accelerator --accelerator-arn <accelerator-arn> --query '{Status:Status,IpAddresses:IpSets,Enabled:Enabled}'`
- Verify accelerator is enabled: `aws globalaccelerator describe-accelerator --accelerator-arn <accelerator-arn> --query 'Accelerator.Enabled'`
- Test connectivity to static IPs: `ping <static-ip-1>` and `nc -zv <static-ip-1> <port>`
- Check DNS resolution of accelerator DNS name: `nslookup <accelerator-dns-name>`

SHOULD:
- Check if BYOIP is in use and its status: `aws globalaccelerator list-byoip-cidrs`
- Verify listeners are configured on expected ports: `aws globalaccelerator list-listeners --accelerator-arn <accelerator-arn>`
- Check CloudWatch for processed bytes: `aws cloudwatch get-metric-statistics --namespace AWS/GlobalAccelerator --metric-name ProcessedBytesIn --dimensions Name=Accelerator,Value=<accelerator-arn> --start-time <start> --end-time <end> --period 300 --statistics Sum`

MAY:
- Check if the accelerator was recently created (propagation may be in progress)
- Verify no network ACLs or firewalls are blocking the anycast IPs at the client side

## Phase 2 — Remediate

MUST:
- Ensure the accelerator is in DEPLOYED status and Enabled is true
- Verify at least one listener with port ranges is configured
- Ensure endpoint groups have healthy endpoints

SHOULD:
- If using BYOIP, ensure CIDR is in ADVERTISING state
- Update DNS records if migrating from standard IPs to BYOIP

MAY:
- Create a new accelerator if IPs need to change (IPs are immutable per accelerator)

## Common Issues

- symptoms: "Static IPs are unreachable"
  diagnosis: "Accelerator is disabled or in IN_PROGRESS state."
  resolution: "Enable the accelerator and wait for DEPLOYED status."

- symptoms: "BYOIP CIDR not advertising"
  diagnosis: "BYOIP provisioning is incomplete or CIDR is in FAILED_ADVERTISING state."
  resolution: "Check BYOIP CIDR status and re-advertise if needed: aws globalaccelerator advertise-byoip-cidr --cidr <cidr>"

- symptoms: "DNS resolves but connection times out"
  diagnosis: "No listeners configured or all endpoints are unhealthy."
  resolution: "Configure listeners with correct port ranges and ensure endpoints are healthy."

## Output Format

```yaml
root_cause: "static_ip_issue — <specific_cause>"
evidence:
  - type: accelerator_status
    content: "<accelerator status and enabled state>"
  - type: ip_addresses
    content: "<static IP details>"
  - type: connectivity_test
    content: "<ping/nc results>"
severity: HIGH
mitigation:
  immediate: "Enable accelerator or fix listener/endpoint configuration"
  long_term: "Monitor accelerator status and endpoint health"
```

## Safety Ratings
```
safety_ratings:
  - "Triage commands (describe/list/get): GREEN — read-only"
  - "Configuration changes: YELLOW — recoverable"
  - "Delete/terminate operations: RED — irreversible"
```

## Escalation Conditions
- "Resource serves production workload"
- "Fix requires deleting or recreating resources"
- "Fix involves security or encryption changes"
- "Blast radius affects multiple dependent services"

## Data Sensitivity
- HIGH: credentials, PII, security configurations
- MEDIUM: resource configurations, metrics
- LOW: status information, error messages

## Prohibited Actions
- Never disable or delete an accelerator serving production traffic without confirming failover
- Never remove endpoint groups without verifying traffic has been drained
- Never modify static IP associations on active accelerators
- Never change routing configurations during active incidents without approval
- Never delete flow logs that may be needed for audit or investigation

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Re-enable previous endpoint group configurations if traffic routing degrades
- Restore traffic dial settings to previous values if performance worsens
- Revert client affinity settings if session persistence breaks

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
