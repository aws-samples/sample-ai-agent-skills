---
title: "G1 — Snowcone-Specific Issues"
description: "Diagnose AWS Snowcone-specific issues"
status: active
severity: MEDIUM
triggers: ["Snowcone error", "Snowcone DataSync", "Snowcone storage", "Snowcone compute"]
owner: devops-agent
objective: "Resolve Snowcone-specific issues including storage limits, DataSync agent, compute constraints, and Wi-Fi connectivity"
context: "Snowcone has 14 TB storage, limited compute (2 vCPUs, 4 GB RAM), supports DataSync agent, and has different capabilities than Snowball Edge."
---

## Phase 1 — Triage

MUST:
- Check device status: `snowballEdge describe-device --endpoint https://<device-ip>:8243 --manifest-file <manifest> --unlock-code <code>`
- Verify device model is Snowcone (SNC1_HDD or SNC1_SSD)
- Check available storage capacity

SHOULD:
- Verify DataSync agent status if using DataSync for transfer
- Check compute resource availability (2 vCPUs, 4 GB RAM limit)

MAY:
- Test Wi-Fi connectivity if using wireless interface
- Review Snowcone-specific service limitations

## Phase 2 — Remediate

MUST:
- Ensure workload fits within Snowcone resource limits (14 TB storage, 2 vCPUs, 4 GB RAM)
- Use DataSync agent for data transfer (recommended for Snowcone)
- Configure network interface correctly (RJ45 or Wi-Fi depending on model)

SHOULD:
- Optimize data for Snowcone's storage capacity
- Use lightweight compute workloads appropriate for Snowcone resources

MAY:
- Configure Wi-Fi settings for wireless data transfer
- Use external power source (USB-C) for portable deployments

## Safety Ratings

```
safety_ratings:
  - "describe-device: GREEN — read-only device status"
  - "DataSync agent configuration: YELLOW — configures data transfer"
  - "start-service: YELLOW — starts services on device"
  - "Wi-Fi configuration: YELLOW — modifies network settings"
```

## Escalation Conditions

- Snowcone storage is insufficient for the data volume
- DataSync agent fails to connect to AWS DataSync service
- Compute workload exceeds Snowcone's 2 vCPU / 4 GB RAM limit
- Wi-Fi connectivity is unreliable for data transfer

## Data Sensitivity

- `snowballEdge describe-device`: MEDIUM — reveals device configuration
- DataSync agent logs: MEDIUM — reveals transfer details
- Wi-Fi configuration: MEDIUM — reveals network credentials

## Prohibited Actions

- NEVER attempt to run Snowball Edge workloads on Snowcone without verifying compatibility
- NEVER exceed Snowcone storage capacity — it causes transfer failures
- NEVER use Snowcone for large-scale data migration (use Snowball Edge instead)
- NEVER store Wi-Fi credentials in plain text

## Phase 3 — Rollback

1. If DataSync agent was misconfigured, reconfigure with correct settings
2. If services were started with wrong parameters, stop and restart
3. Verify data integrity on device after any service changes
4. Restore network configuration if Wi-Fi settings were changed
5. Document Snowcone-specific configuration for the deployment

## Common Issues

- symptoms: "DataSync agent cannot connect to AWS DataSync service"
  diagnosis: "Snowcone does not have internet connectivity or DataSync endpoint is not reachable."
  resolution: "Verify internet connectivity. Ensure DNS resolves DataSync endpoints. Allow outbound HTTPS to datasync.*.amazonaws.com."

## Output Format

```yaml
root_cause: "snowcone_specific — <specific_cause>"
evidence:
  - type: snowcone_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix Snowcone issue"
  long_term: "Right-size device selection"
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
