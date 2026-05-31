---
title: "D2 — Network Function Groups"
description: "Diagnose network function group (NFG) configuration and service insertion issues"
status: active
severity: CRITICAL
triggers: ["network function group error", "service insertion failed", "NFG not working", "firewall insertion issue", "traffic inspection failed"]
owner: devops-agent
objective: "Resolve network function group configuration and service insertion issues"
context: "Network function groups enable service insertion (firewalls, IDS/IPS) in the core network traffic path. NFGs require specific policy sections: network-function-groups and service-insertion."
---

## Phase 1 — Triage

MUST:
- Get core network policy to review NFG configuration: `aws networkmanager get-core-network-policy --core-network-id <id>`
- Verify the network-function-groups section exists in the policy
- Check the service-insertion section for send-via or send-to configuration
- List attachments to verify NFG appliance attachments: `aws networkmanager list-attachments --core-network-id <id>`

SHOULD:
- Verify the appliance attachment (Connect attachment) is in AVAILABLE state
- Check if the appliance (firewall/IDS) is healthy and processing traffic
- Review segment-actions for correct NFG routing

MAY:
- Check VPC Flow Logs on the appliance VPC for traffic visibility
- Verify the Connect attachment peer is established

## Phase 2 — Remediate

MUST:
- Add or fix the network-function-groups section in the core network policy
- Configure service-insertion with correct send-via or send-to rules
- Ensure the appliance attachment is assigned to the correct NFG via attachment-policies

SHOULD:
- Submit and execute the corrected policy
- Verify traffic is flowing through the appliance after policy execution

MAY:
- Set up health checks for the network function appliance
- Implement redundant appliances across AZs

## Common Issues

- symptoms: "Traffic bypasses the firewall appliance"
  diagnosis: "service-insertion section missing or send-via not configured for the affected segments."
  resolution: "Add service-insertion with send-via rules specifying the NFG for the source and destination segments."

- symptoms: "Connect attachment for appliance in FAILED state"
  diagnosis: "Connect attachment requires a transport attachment (VPC attachment) that is AVAILABLE."
  resolution: "Ensure the transport VPC attachment is AVAILABLE, then recreate the Connect attachment."

## Safety Ratings

```
safety_ratings:
  - "get-core-network-policy: GREEN — read-only policy retrieval"
  - "put-core-network-policy (NFG changes): RED — affects traffic inspection path"
  - "execute-core-network-change-set (NFG): RED — activates service insertion changes"
  - "create-connect-attachment: YELLOW — creates appliance connectivity"
```

## Escalation Conditions

- NFG misconfiguration causes traffic to bypass security inspection in production
- Service insertion causes traffic black-holing (traffic sent to NFG but not returned)
- Appliance health check failures causing intermittent traffic drops
- NFG configuration changes affect compliance-mandated traffic inspection

## Data Sensitivity

- `aws networkmanager get-core-network-policy`: MEDIUM — reveals security inspection architecture
- `aws networkmanager list-attachments`: LOW — returns attachment metadata
- VPC Flow Logs on appliance VPC: HIGH — reveals inspected traffic patterns

## Prohibited Actions

- NEVER remove service-insertion rules in production without confirming traffic will not bypass inspection
- NEVER delete NFG appliance attachments without failover to a backup appliance
- NEVER modify NFG configuration without security team approval
- NEVER assume traffic is being inspected just because the NFG is configured — verify with flow logs

## Phase 3 — Rollback

1. If NFG changes caused traffic bypass, immediately rollback the core network policy to the previous version
2. Execute the rollback policy to restore service insertion rules
3. Verify traffic is flowing through the appliance using VPC Flow Logs
4. If the appliance attachment was deleted, recreate it and update the policy
5. Document the incident and notify the security team immediately

## Output Format

```yaml
root_cause: "network_function_group — <specific_cause>"
evidence:
  - type: nfg_config
    content: "<NFG and service-insertion details>"
severity: CRITICAL
mitigation:
  immediate: "Fix NFG configuration and verify traffic inspection"
  long_term: "Implement NFG health monitoring and automated failover"
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
