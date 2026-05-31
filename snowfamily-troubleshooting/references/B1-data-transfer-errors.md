---
title: "B1 — Data Transfer Errors"
description: "Diagnose AWS Snow Family data transfer failures"
status: active
severity: HIGH
triggers: ["data transfer failed", "copy to Snowball failed", "transfer error", "slow transfer"]
owner: devops-agent
objective: "Resolve data transfer issues including S3 adapter errors, performance, and file compatibility"
context: "Data transfer uses S3 adapter or NFS interface. Performance depends on network interface, file sizes, and transfer method."
---

## Phase 1 — Triage

MUST:
- Check device status: `snowballEdge describe-device --endpoint https://<device-ip>:8243 --manifest-file <manifest> --unlock-code <code>`
- Verify S3 adapter service is running: `snowballEdge describe-service --service-id s3 --endpoint https://<device-ip>:8243 --manifest-file <manifest> --unlock-code <code>`
- Test S3 adapter connectivity: `aws s3 ls --endpoint https://<device-ip>:8443 --profile snowball`

SHOULD:
- Check available storage on device
- Verify network interface speed and configuration

MAY:
- Monitor transfer throughput using system tools
- Check for file name or size compatibility issues

## Phase 2 — Remediate

MUST:
- Restart S3 adapter service if not running: `snowballEdge start-service --service-id s3`
- Fix S3 endpoint configuration in AWS CLI profile
- Ensure sufficient storage space on device

SHOULD:
- Use parallel transfers for better throughput: `aws s3 cp --recursive --endpoint <endpoint>`
- Batch small files into archives before transfer

MAY:
- Switch to NFS interface for file-based transfers
- Use the fastest available network interface (QSFP28 > SFP+ > RJ45)

## Safety Ratings

```
safety_ratings:
  - "describe-device: GREEN — read-only device status"
  - "describe-service: GREEN — read-only service status"
  - "s3 cp to device: YELLOW — writes data to device storage"
  - "start-service: YELLOW — starts a service on the device"
```

## Escalation Conditions

- S3 adapter service crashes repeatedly after restart
- Transfer speeds are significantly below expected throughput for the interface
- Device storage is full and data transfer is incomplete
- File corruption detected after transfer

## Data Sensitivity

- `snowballEdge describe-device`: MEDIUM — reveals device configuration
- `aws s3 ls (device endpoint)`: MEDIUM — reveals data on device
- Transfer logs: HIGH — may reveal file names and data patterns

## Prohibited Actions

- NEVER delete data from the device without confirming it exists in the source
- NEVER transfer sensitive data without verifying encryption is active
- NEVER ignore transfer errors — they may indicate data corruption
- NEVER exceed device storage capacity — it causes transfer failures

## Phase 3 — Rollback

1. If incorrect data was transferred, delete it from the device: `aws s3 rm --recursive --endpoint <endpoint> s3://<bucket>/<prefix>`
2. If S3 adapter configuration was changed, restart the service
3. Verify data integrity by comparing checksums between source and device
4. If transfer is incomplete, resume from the last successful file
5. Document transfer issues and throughput metrics

## Common Issues

- symptoms: "S3 PutObject returns 'NoSuchBucket' on device endpoint"
  diagnosis: "Bucket was not created on the device S3 adapter."
  resolution: "Create the bucket on the device: aws s3 mb s3://<bucket> --endpoint https://<device-ip>:8443"

## Output Format

```yaml
root_cause: "data_transfer — <specific_cause>"
evidence:
  - type: transfer_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix transfer issue"
  long_term: "Optimize transfer pipeline"
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
