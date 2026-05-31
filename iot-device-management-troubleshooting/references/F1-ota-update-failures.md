---
title: "F1 — OTA Update Failures"
description: "Diagnose over-the-air firmware update creation, delivery, and execution failures"
status: active
severity: CRITICAL
triggers: ["OTA update failed", "firmware update error", "code signing failed", "OTA job not executing", "update stuck"]
owner: devops-agent
objective: "Resolve OTA update failures and ensure firmware is delivered to target devices"
context: "OTA updates use IoT jobs with code signing to deliver firmware. Requires S3 bucket for firmware, code-signing profile, IAM role, and device-side OTA agent. Failures can occur at creation, signing, delivery, or device execution."
---

## Phase 1 — Triage

MUST:
- List OTA updates: `aws iot list-ota-updates --query 'otaUpdates[*].{Id:otaUpdateId,Status:otaUpdateStatus,CreationDate:creationDate}'`
- Describe the failing update: `aws iot get-ota-update --ota-update-id <id>`
- Check the underlying job status: `aws iot describe-job --job-id <ota-job-id>`

SHOULD:
- Verify code-signing profile: `aws signer list-signing-profiles --query 'profiles[*].{Name:profileName,Status:status,Platform:platformId}'`
- Check S3 firmware file exists and is accessible: `aws s3 ls s3://<bucket>/<firmware-key>`
- Verify OTA service role permissions: `aws iam get-role --role-name <ota-role>`

MAY:
- Check signing job status: `aws signer list-signing-jobs --query 'jobs[*].{Id:jobId,Status:status,ProfileName:profileName}'`
- Review CloudWatch Logs for OTA errors
- Check device-side OTA agent logs for download or apply failures

## Phase 2 — Remediate

MUST:
- Ensure S3 bucket allows IoT service to read firmware file (bucket policy or IAM role)
- Verify code-signing profile is active and uses correct signing algorithm for target platform
- Ensure OTA service role has permissions: iot:CreateJob, s3:GetObject, signer:StartSigningJob, signer:DescribeSigningJob

SHOULD:
- Verify firmware file is not corrupted (check S3 object checksum)
- Ensure target devices have OTA agent running and connected
- Configure abort criteria to stop rollout if failure rate exceeds threshold

MAY:
- Test OTA update on a single device before fleet-wide deployment
- Enable detailed IoT logging for OTA-related events
- Set up CloudWatch alarms for OTA job failure metrics

## Safety Ratings

```
safety_ratings:
  - "list-ota-updates: GREEN — read-only enumeration"
  - "get-ota-update: GREEN — read-only inspection"
  - "create-ota-update: RED — deploys firmware to devices, irreversible on devices without bootloader rollback"
  - "delete-ota-update: YELLOW — removes OTA update record, does not revert firmware"
```

## Escalation Conditions

- OTA update bricks devices (firmware causes boot failure) — requires physical intervention
- Code-signing key compromise requires immediate rotation and re-signing
- OTA update deployed to wrong device group affecting production fleet
- Firmware vulnerability discovered after partial rollout

## Data Sensitivity

- `aws iot get-ota-update`: MEDIUM — contains firmware location and signing details
- `aws s3 ls` (firmware bucket): LOW — returns object metadata
- `aws signer list-signing-profiles`: MEDIUM — contains signing configuration
- Code-signing private keys: CRITICAL — must never be exposed

## Prohibited Actions

- NEVER deploy OTA updates to production fleet without testing on a small group first
- NEVER skip code signing — unsigned firmware bypasses integrity verification
- NEVER delete the S3 firmware file while an OTA update is in progress
- NEVER ignore OTA failures on even a small percentage of devices — investigate root cause before continuing

## Phase 3 — Rollback

1. Cancel the OTA update to stop further deployments: `aws iot delete-ota-update --ota-update-id <id> --force-delete`
2. If devices received bad firmware, deploy a rollback OTA update with previous firmware version
3. For devices that cannot receive OTA (bricked), plan physical recovery procedure
4. Verify remaining fleet is on correct firmware version via device shadow or fleet indexing
5. Document the incident including affected device count and firmware versions

## Common Issues

- symptoms: "OTA update status is CREATE_FAILED"
  diagnosis: "Code-signing profile is invalid, S3 file not found, or IAM role missing required permissions."
  resolution: "Verify signing profile is active. Confirm S3 object exists. Check IAM role has iot:CreateJob, s3:GetObject, and signer permissions."

- symptoms: "OTA job created but devices show FAILED execution"
  diagnosis: "Device OTA agent cannot download firmware (network issue) or firmware signature verification fails on device."
  resolution: "Check device connectivity. Verify signing algorithm matches device expectations. Check device has sufficient storage for firmware."

## Output Format

```yaml
root_cause: "ota_update — <specific_cause>"
evidence:
  - type: ota_error
    content: "<error details>"
severity: CRITICAL
mitigation:
  immediate: "Cancel failing OTA and investigate"
  long_term: "Implement staged rollout with abort criteria and device health monitoring"
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
  - command: "describe-instances"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-console-output"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "ssm send-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound security group rules as a fix"
  - "NEVER suggest disabling instance metadata service"
  - "NEVER terminate instances without confirmation"
