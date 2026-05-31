---
title: "C1 — Global Accelerator Client Affinity Issues"
description: "Diagnose client affinity (session persistence) problems"
status: active
severity: MEDIUM
triggers:
  - "client affinity"
  - "session persistence"
  - "sticky sessions"
  - "SOURCE_IP"
owner: devops-agent
objective: "Resolve client affinity configuration issues in Global Accelerator"
context: "Client affinity is configured per listener. SOURCE_IP affinity routes all connections from the same client IP to the same endpoint. Default is NONE. Affinity applies to all endpoint groups under the listener."
---

## Phase 1 — Triage

MUST:
- Check listener client affinity setting: `aws globalaccelerator describe-listener --listener-arn <listener-arn> --query '{ClientAffinity:Listener.ClientAffinity,Protocol:Listener.Protocol,PortRanges:Listener.PortRanges}'`
- List all listeners and their affinity settings: `aws globalaccelerator list-listeners --accelerator-arn <accelerator-arn>`
- Check endpoint health (affinity only works with healthy endpoints): `aws globalaccelerator describe-endpoint-group --endpoint-group-arn <endpoint-group-arn> --query 'EndpointGroup.EndpointDescriptions[*].{EndpointId:EndpointId,HealthState:HealthState}'`
- Verify traffic is reaching the expected endpoint by checking flow logs or application logs

SHOULD:
- Test from the same client IP to verify affinity behavior
- Check if NAT or proxy is changing the client source IP
- Verify endpoint weights are not set to 0 for the expected endpoint

MAY:
- Enable flow logs to trace client IP to endpoint mapping
- Check if the endpoint was recently added (new endpoints may not have existing affinity mappings)

## Phase 2 — Remediate

MUST:
- Set client affinity on the listener: `aws globalaccelerator update-listener --listener-arn <listener-arn> --client-affinity SOURCE_IP`
- Or disable affinity: `aws globalaccelerator update-listener --listener-arn <listener-arn> --client-affinity NONE`

SHOULD:
- Ensure all endpoints in the target endpoint group are healthy
- Understand that affinity applies to ALL endpoint groups under the listener

MAY:
- Use application-level session management if GA affinity is insufficient

## Common Issues

- symptoms: "Requests from same client going to different endpoints"
  diagnosis: "Client affinity is set to NONE on the listener."
  resolution: "Update listener client affinity to SOURCE_IP."

- symptoms: "Client affinity set but still seeing different endpoints"
  diagnosis: "Client is behind NAT/proxy with different source IPs, or endpoint became unhealthy."
  resolution: "Check if client source IP is consistent. Verify endpoint health."

- symptoms: "Cannot set different affinity per endpoint group"
  diagnosis: "Client affinity is a listener-level setting, not per-endpoint-group."
  resolution: "Create separate listeners with different port ranges if different affinity is needed."

## Output Format

```yaml
root_cause: "client_affinity — <specific_cause>"
evidence:
  - type: listener_config
    content: "<listener affinity setting>"
  - type: endpoint_health
    content: "<endpoint health status>"
  - type: client_ip
    content: "<client source IP analysis>"
severity: MEDIUM
mitigation:
  immediate: "Configure client affinity on the listener"
  long_term: "Implement application-level session management as backup"
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
