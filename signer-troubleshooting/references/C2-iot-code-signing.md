---
title: "C2 — IoT Code Signing"
description: "Diagnose IoT code signing and firmware signing issues"
status: active
severity: HIGH
triggers: ["IoT signing failed", "firmware signing error", "IoT OTA update signing"]
owner: devops-agent
objective: "Resolve IoT code signing issues for firmware updates"
context: "IoT code signing uses specific platforms (AmazonFreeRTOS, AWSIoTDeviceManagement). Signed firmware is used for OTA updates. Different hardware targets require different signing platforms."
---

## Phase 1 — Triage

MUST:
- List IoT signing platforms: `aws signer list-signing-platforms --query 'platforms[?category==\`AWS\`]'`
- Check signing profile for IoT: `aws signer get-signing-profile --profile-name <profile-name>`
- Verify the signing job status: `aws signer describe-signing-job --job-id <job-id>`

SHOULD:
- Verify the platform ID matches the target hardware
- Check if the firmware binary format is correct for the platform

MAY:
- Review IoT OTA update job status: `aws iot describe-job --job-id <iot-job-id>`
- Check IoT device logs for firmware update failures

## Phase 2 — Remediate

MUST:
- Use the correct IoT signing platform for the target hardware
- Create IoT-specific signing profile: `aws signer put-signing-profile --profile-name <name> --platform-id AWSIoTDeviceManagement-SHA256-ECDSA`
- Ensure the firmware binary is in the correct format for the platform

SHOULD:
- Verify the signed firmware can be deployed to test devices before production
- Check IoT thing group targeting for OTA updates

MAY:
- Set up IoT OTA update monitoring
- Configure signing notifications for IoT firmware signing jobs

## Common Issues

- symptoms: "IoT signing job fails with 'Invalid platform' error"
  diagnosis: "Using a Lambda signing platform for IoT firmware."
  resolution: "Use an IoT-specific platform: AWSIoTDeviceManagement-SHA256-ECDSA or AmazonFreeRTOS-TI-CC3220SF."

- symptoms: "Signed firmware fails to install on IoT device"
  diagnosis: "Firmware binary format mismatch or wrong signing platform for the hardware."
  resolution: "Verify the signing platform matches the target hardware. Check firmware binary format."

## Safety Ratings

```
safety_ratings:
  - "list-signing-platforms: GREEN — read-only platform listing"
  - "get-signing-profile: GREEN — read-only profile details"
  - "start-signing-job for IoT: YELLOW — signs firmware for deployment"
  - "iot create-ota-update: RED — deploys firmware to IoT devices"
```

## Escalation Conditions

- IoT firmware signing fails for a critical security patch
- Signed firmware causes device failures during OTA update
- Target hardware platform not supported by available signing platforms
- Large-scale OTA update blocked by signing issues

## Data Sensitivity

- `aws signer list-signing-platforms`: LOW — public platform information
- IoT firmware binaries: HIGH — contain device firmware code
- Signed firmware: HIGH — contains signed device firmware
- IoT device fleet information: MEDIUM — reveals device inventory

## Prohibited Actions

- NEVER deploy unsigned firmware to production IoT devices
- NEVER use Lambda signing platforms for IoT firmware
- NEVER push OTA updates to all devices simultaneously without staged rollout
- NEVER ignore firmware signing failures — they may indicate binary corruption

## Phase 3 — Rollback

1. If an OTA update was started with incorrectly signed firmware, cancel it: `aws iot cancel-job --job-id <job-id>`
2. If a signing profile was created with wrong platform, cancel it
3. Prepare a rollback firmware image signed with the correct profile
4. Deploy rollback firmware to affected devices
5. Document the firmware signing issue and resolution

## Output Format

```yaml
root_cause: "iot_code_signing — <specific_cause>"
evidence:
  - type: iot_signing_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix IoT signing configuration"
  long_term: "Standardize IoT firmware signing pipeline"
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
