# B2 — vCenter Integration Issues

## Symptoms
- vCenter connection fails
- VMs not discovered from vCenter
- Partial VM discovery

## Triage Commands

```bash
aws discovery list-configurations --configuration-type SERVER --max-results 50
aws discovery get-discovery-summary
```

## Common Causes

1. **vCenter URL wrong** — Must use correct vCenter Server URL
2. **SSL certificate** — Self-signed cert not trusted by collector
3. **vCenter permissions** — User needs read access to inventory
4. **Firewall blocking** — Collector can't reach vCenter on port 443
5. **Large inventory** — Discovery takes time for large environments

## Resolution

- Verify vCenter URL: `https://<vcenter-host>/sdk`
- Accept self-signed certificates in collector configuration
- Grant vCenter user read-only role at datacenter level
- Allow collector to reach vCenter on port 443
- For large environments, allow several hours for initial discovery
- Check collector logs via web UI for specific errors


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Verify vCenter URL | GREEN | Read-only check |
| Accept self-signed certificates | YELLOW | Security posture change |
| Grant vCenter read-only role | YELLOW | Permission change in vCenter |
| Allow collector to reach vCenter | YELLOW | Network/firewall change |
| Wait for large environment discovery | GREEN | Passive wait |

## Escalation Conditions

- Discovery agent deployed in production environment

## Data Sensitivity

- **vCenter URL and credentials**: Infrastructure access details — CONFIDENTIAL
- **VM inventory data**: Server names, IPs, configurations — INTERNAL
- **SSL certificates**: Certificate trust decisions — INTERNAL

## Prohibited Actions

- NEVER deploy discovery agents without change management approval
- NEVER store IAM access keys in plain text on shared systems
- NEVER modify agent configuration on production servers without a rollback plan

## Phase 3 — Rollback

1. Remove vCenter credentials from the collector configuration
2. Revoke the vCenter user account created for discovery
3. Re-enable SSL certificate validation if it was relaxed
4. Close firewall rules opened for collector-to-vCenter communication
5. Verify discovery has stopped: `aws discovery get-discovery-summary`

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
