---
title: "A1 — Gateway Deployment Failures"
description: "Diagnose Backup Gateway deployment and activation failures"
status: active
severity: CRITICAL
triggers: ["gateway deployment failed", "cannot activate gateway", "OVA deployment error", "gateway not appearing"]
owner: devops-agent
objective: "Resolve gateway deployment and activation issues"
context: "Gateway is an OVA deployed on VMware ESXi. Requires network connectivity to AWS endpoints on port 443."
---

## Phase 1 — Triage

MUST:
- Check if gateway appears in AWS: `aws backup-gateway list-gateways`
- Verify the gateway VM is powered on in vCenter
- Test network connectivity from gateway to AWS endpoints: verify DNS resolution and port 443 access

SHOULD:
- Check gateway VM resource allocation (CPU, memory, disk) against minimum requirements
- Verify NTP synchronization on the gateway VM

MAY:
- Review vCenter events for OVA deployment errors
- Check ESXi host resource availability

## Phase 2 — Remediate

MUST:
- Ensure gateway VM meets minimum resource requirements (4 vCPU, 16 GB RAM, 80 GB disk)
- Configure DNS to resolve AWS service endpoints
- Open firewall rules for port 443 to AWS endpoints
- Activate the gateway with the activation key: `aws backup-gateway create-gateway --activation-key <key> --gateway-display-name <name> --gateway-type BACKUP_VM`

SHOULD:
- Configure NTP for accurate time synchronization
- Set up static IP if DHCP is unreliable

MAY:
- Configure proxy settings if direct internet access is not available
- Deploy in a dedicated management VLAN

## Safety Ratings

```
safety_ratings:
  - "backup-gateway list-gateways: GREEN — read-only gateway enumeration"
  - "backup-gateway create-gateway: YELLOW — creates gateway resource in AWS"
  - "OVA deployment: YELLOW — creates VM in VMware environment"
  - "firewall rule changes: RED — modifies network security"
```

## Escalation Conditions

- Gateway VM deploys but cannot reach AWS endpoints after firewall changes
- Activation key expires before gateway can be activated (valid for limited time)
- ESXi host lacks resources to run the gateway VM
- Network team cannot open required firewall rules

## Data Sensitivity

- `aws backup-gateway list-gateways`: LOW — returns gateway metadata
- Activation key: HIGH — grants gateway registration to the AWS account
- Network configuration: MEDIUM — reveals network topology and firewall rules

## Prohibited Actions

- NEVER share the activation key outside the deployment team
- NEVER deploy the gateway on an ESXi host with insufficient resources
- NEVER skip NTP configuration — time skew causes authentication failures
- NEVER open broader firewall rules than required (port 443 to AWS endpoints only)

## Phase 3 — Rollback

1. If gateway was created in AWS but deployment failed, delete it: `aws backup-gateway delete-gateway --gateway-arn <arn>`
2. If the OVA was deployed incorrectly, delete the VM from vCenter and redeploy
3. Revert firewall rule changes if they were made during troubleshooting
4. Generate a new activation key if the previous one expired
5. Document the deployment configuration for future reference

## Common Issues

- symptoms: "Gateway VM is running but does not appear in AWS console"
  diagnosis: "Gateway cannot reach AWS endpoints — DNS, firewall, or proxy misconfiguration."
  resolution: "Verify DNS resolves AWS endpoints. Check firewall allows port 443 outbound. Configure proxy if required."

## Output Format

```yaml
root_cause: "gateway_deployment — <specific_cause>"
evidence:
  - type: deployment_error
    content: "<error details>"
severity: CRITICAL
mitigation:
  immediate: "Fix deployment issue"
  long_term: "Document deployment prerequisites and network requirements"
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
