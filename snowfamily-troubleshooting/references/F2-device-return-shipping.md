---
title: "F2 — Device Return Shipping"
description: "Diagnose Snow Family device return and shipping issues"
status: active
severity: MEDIUM
triggers: ["return shipping", "shipping label", "device return", "late return", "import to S3"]
owner: devops-agent
objective: "Resolve device return issues including shipping labels, tracking, return deadlines, and S3 import"
context: "Devices must be returned within the allotted time. E Ink shipping label updates automatically. Data imports to S3 after return."
---

## Phase 1 — Triage

MUST:
- Check job status: `aws snowball describe-job --job-id <job-id> --query 'JobMetadata.{State:JobState,ShippingDetails:ShippingDetails}'`
- Verify return shipping label is displayed on device E Ink display
- Confirm all data transfer is complete before returning

SHOULD:
- Check return deadline to avoid late fees
- Verify target S3 bucket exists and has correct permissions

MAY:
- Track return shipment: `aws snowball get-job-manifest --job-id <job-id>`
- Review job notification settings for import status updates

## Phase 2 — Remediate

MUST:
- Power off device properly before shipping
- Ensure E Ink shipping label shows return address (not delivery address)
- Verify S3 bucket and KMS key are available for data import

SHOULD:
- Schedule pickup with the carrier shown on the shipping label
- Notify AWS Support if shipping label is not displaying correctly

MAY:
- Set up SNS notifications for job status changes
- Verify S3 bucket lifecycle policies won't delete imported data

## Safety Ratings

```
safety_ratings:
  - "describe-job: GREEN — read-only job details"
  - "Power off device: YELLOW — stops all services and data access"
  - "Ship device: YELLOW — physical device leaves your control"
  - "S3 bucket policy changes: YELLOW — modifies data access"
```

## Escalation Conditions

- E Ink shipping label is blank or shows incorrect address
- Return deadline is approaching and data transfer is incomplete
- Device was damaged during use and needs special handling
- S3 import fails after device is returned to AWS

## Data Sensitivity

- `aws snowball describe-job`: MEDIUM — reveals shipping details and addresses
- Shipping label: LOW — contains carrier routing information
- S3 import status: MEDIUM — reveals data transfer completion

## Prohibited Actions

- NEVER ship a device without verifying the E Ink label shows the return address
- NEVER return a device with the power cable still connected
- NEVER delete the target S3 bucket before data import completes
- NEVER ignore return deadlines — late returns incur daily charges

## Phase 3 — Rollback

1. If device was shipped to wrong address, contact the carrier immediately
2. If S3 bucket was deleted before import, recreate it with the same name
3. If KMS key was disabled, re-enable it before import begins
4. Contact AWS Support if import fails after device return
5. Document the return process and any issues for future jobs

## Common Issues

- symptoms: "Job status stuck in 'InTransitToAWS' for extended period"
  diagnosis: "Device is in transit or carrier tracking is delayed."
  resolution: "Check carrier tracking number from describe-job output. Contact carrier if tracking shows delivery but job status hasn't updated."

## Output Format

```yaml
root_cause: "return_shipping — <specific_cause>"
evidence:
  - type: shipping_detail
    content: "<shipping status>"
severity: MEDIUM
mitigation:
  immediate: "Resolve shipping issue"
  long_term: "Track return deadlines"
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
