---
title: "B1 — Hypervisor Connection Errors"
description: "Diagnose hypervisor connection errors between gateway and vCenter/ESXi"
status: active
severity: HIGH
triggers: ["hypervisor connection failed", "cannot connect to vCenter", "hypervisor offline", "ESXi connection error"]
owner: devops-agent
objective: "Resolve hypervisor connection issues between the gateway and VMware environment"
context: "Gateway connects to vCenter or ESXi hosts to discover and back up VMs. Connection requires valid credentials and network access."
---

## Phase 1 — Triage

MUST:
- Check hypervisor status: `aws backup-gateway list-hypervisors --query 'Hypervisors[*].{Name:Name,Arn:HypervisorArn,Host:Host,State:State}'`
- Get hypervisor details: `aws backup-gateway get-hypervisor --hypervisor-arn <hypervisor-arn>`
- Verify network connectivity from gateway to vCenter host on port 443

SHOULD:
- Check if vCenter credentials have been rotated recently
- Verify vCenter/ESXi is operational and accessible

MAY:
- Review CloudTrail for hypervisor-related events
- Check vCenter event logs for connection attempts

## Phase 2 — Remediate

MUST:
- Fix network connectivity between gateway and vCenter
- Update credentials if they were rotated: `aws backup-gateway update-hypervisor --hypervisor-arn <arn> --username <user> --password <pass>`
- Verify vCenter SSL certificate is trusted by the gateway

SHOULD:
- Test vCenter connectivity from the gateway network
- Verify the vCenter user has required permissions (read-only + snapshot management)

MAY:
- Configure a dedicated vCenter service account for Backup Gateway
- Set up monitoring for hypervisor connection status

## Safety Ratings

```
safety_ratings:
  - "backup-gateway list-hypervisors: GREEN — read-only hypervisor listing"
  - "backup-gateway get-hypervisor: GREEN — read-only hypervisor details"
  - "backup-gateway update-hypervisor: YELLOW — updates hypervisor credentials"
  - "network connectivity testing: GREEN — read-only network diagnostics"
```

## Escalation Conditions

- vCenter is unreachable from the gateway network
- vCenter credentials require VMware admin to reset
- SSL certificate changes on vCenter break the connection
- Multiple hypervisor connections fail simultaneously

## Data Sensitivity

- `aws backup-gateway list-hypervisors`: LOW — returns hypervisor metadata
- `aws backup-gateway get-hypervisor`: MEDIUM — reveals vCenter host addresses
- vCenter credentials: CRITICAL — provides access to the VMware environment
- `aws backup-gateway update-hypervisor`: HIGH — modifies stored credentials

## Prohibited Actions

- NEVER store vCenter credentials in plain text or logs
- NEVER use vCenter administrator accounts — use a dedicated service account
- NEVER modify vCenter configuration to fix gateway connection issues
- NEVER skip SSL certificate validation as a troubleshooting step

## Phase 3 — Rollback

1. If credentials were updated incorrectly, update again with correct credentials
2. If hypervisor was deleted, re-add it: `aws backup-gateway import-hypervisor-configuration --host <host> --username <user> --password <pass> --name <name>`
3. Verify VM discovery resumes after hypervisor connection is restored
4. Check that backup schedules are not disrupted
5. Document the hypervisor connection configuration

## Common Issues

- symptoms: "Hypervisor state shows as OFFLINE"
  diagnosis: "Gateway cannot reach vCenter — network issue or credential expiry."
  resolution: "Verify network path from gateway to vCenter on port 443. If credentials were rotated, update them with update-hypervisor."

## Output Format

```yaml
root_cause: "hypervisor_connection — <specific_cause>"
evidence:
  - type: connection_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Restore hypervisor connection"
  long_term: "Set up credential rotation alerts and connection monitoring"
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
  - "NEVER suggest disabling vault lock"
  - "NEVER suggest deleting backup plans without review"
  - "NEVER suggest removing cross-account backup copies"
